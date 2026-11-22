// GitOps flow: build the image (docker-in-docker), push tags <BUILD_NUMBER> + latest,
// render the Helm chart, and commit the rendered manifests to
//   https://github.com/elevy99927-demos/gitops  ->  systems/<DEPARTMENT>/<ENV>/<appname>.yaml
// Argo CD (ApplicationSet/systems-manifests.yaml) deploys that file.

properties([
  parameters([
    choice(name: 'ENV', choices: ['dev', 'qa', 'prd'], description: 'Environment folder: systems/<DEPARTMENT>/<ENV>/'),
    string(name: 'DEPARTMENT', defaultValue: 'hr', description: 'Department folder: hr, finance, legal, digital, devops'),
    booleanParam(name: 'SERVICE_MONITOR', defaultValue: true, description: 'Render the ServiceMonitor (needs the Prometheus Operator CRDs in the cluster)'),
    choice(name: 'DIND_STORAGE_DRIVER', choices: ['overlay2', 'vfs'], description: 'dind storage driver: overlay2 (fast). Use vfs only if overlay2 fails on your nodes (slow, uses more disk)')
  ])
])

def user          = "hillel456"                             // Docker Hub user
def appname       = "RecruitHub"
def appimage      = "${user}/${appname.toLowerCase()}"      // docker.io/hillel456/recruithub
def apptag        = "${env.BUILD_NUMBER}"
def envName       = params.ENV ?: 'dev'
def department    = params.DEPARTMENT ?: 'hr'
def serviceMon    = params.SERVICE_MONITOR ? 'true' : 'false'
def storageDrv    = params.DIND_STORAGE_DRIVER ?: 'overlay2'

def gitopsRepo    = "github.com/elevy99927-demos/gitops.git"
def gitopsFile    = "systems/${department}/${envName}/${appname}.yaml"
def dashboardFile = "dashboards/${appname}.json"

podTemplate(containers: [
      containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
      containerTemplate(name: 'deployer', image: 'elevy99927/k8s-deployer:latest', command: 'cat', ttyEnabled: true),
      containerTemplate(name: 'docker', image: 'docker:27-dind', privileged: true,
                        args: "--storage-driver=${storageDrv}",
                        envVars: [envVar(key: 'DOCKER_TLS_CERTDIR', value: '')],
                        resourceRequestCpu: '500m', resourceRequestMemory: '512Mi',
                        resourceLimitMemory: '2Gi')
  ],
  volumes: [
      emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
  ])  {
    node(POD_LABEL) {
        stage('checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('build') {
            container('docker') {
                sh '''
                    for i in $(seq 1 30); do docker info >/dev/null 2>&1 && exit 0; sleep 2; done
                    echo "docker daemon did not start"; docker info; exit 1
                '''
                withCredentials([string(credentialsId: 'docker-creds', variable: 'DOCKER_TOKEN')]) {
                    sh "echo \"\$DOCKER_TOKEN\" | docker login -u \"${user}\" --password-stdin"
                }
                echo "Building ${appimage}:${apptag}"
                sh "docker build --network=host -t ${appimage}:${apptag} -t ${appimage}:latest \"${env.WORKSPACE}\""
            }
        }

        stage('push') {
            container('docker') {
                sh """
                    set -e
                    docker push ${appimage}:${apptag}
                    docker push ${appimage}:latest
                    docker logout
                """
            }
        }

        stage('deploy') {
            container('deployer') {
                echo "Rendering chart into ${gitopsRepo} -> ${gitopsFile}"
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                sh """#!/bin/sh
                    set -e
                    command -v git >/dev/null || apk add --no-cache git

                    rm -rf gitops
                    git clone https://\${GIT_USER}:\${GIT_TOKEN}@${gitopsRepo} gitops
                    cd gitops
                    git config user.name  "Jenkins (${appname})"
                    git config user.email "elevy99927-demos@users.noreply.github.com"

                    helm template ${appname.toLowerCase()} "${env.WORKSPACE}/chart" \\
                        --set image.repository=${appimage} \\
                        --set image.tag=${apptag} \\
                        --set serviceMonitor.enabled=${serviceMon} \\
                        > ${gitopsFile}

                    git add ${gitopsFile}
                    if git diff --cached --quiet; then
                        echo "No changes in ${gitopsFile} - nothing to deploy"
                        exit 0
                    fi
                    git commit -m "Deploy ${appimage}:${apptag} to ${department}/${envName} (build ${env.BUILD_NUMBER})"

                    for i in 1 2 3 4 5; do
                        if git push origin HEAD:main; then exit 0; fi
                        echo "push rejected, rebasing (attempt \$i)"
                        git pull --rebase origin main
                        sleep \$i
                    done
                    echo "push failed after 5 attempts"; exit 1
                """
                }
            }
        }

        stage('add-dashboard') {
            container('deployer') {
                echo "Publishing Grafana dashboard into ${gitopsRepo} -> ${dashboardFile}"
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                sh """#!/bin/sh
                    set -e
                    command -v git >/dev/null || apk add --no-cache git

                    rm -rf gitops
                    git clone https://\${GIT_USER}:\${GIT_TOKEN}@${gitopsRepo} gitops
                    cd gitops
                    git config user.name  "Jenkins (${appname})"
                    git config user.email "elevy99927-demos@users.noreply.github.com"

                    mkdir -p dashboards
                    sed -e 's/"Python App Metrics"/"${appname} Metrics"/g' \\
                        -e 's/"python-app-metrics"/"${appname.toLowerCase()}-metrics"/g' \\
                        -e 's/\${DS_PROMETHEUS}/prometheus/g' \\
                        "${env.WORKSPACE}/grafana-dashboard.json" > ${dashboardFile}

                    git add ${dashboardFile}
                    if git diff --cached --quiet; then
                        echo "No changes in ${dashboardFile} - dashboard already up to date"
                        exit 0
                    fi
                    git commit -m "Add/update Grafana dashboard for ${appname} (build ${env.BUILD_NUMBER})"

                    for i in 1 2 3 4 5; do
                        if git push origin HEAD:main; then exit 0; fi
                        echo "push rejected, rebasing (attempt \$i)"
                        git pull --rebase origin main
                        sleep \$i
                    done
                    echo "push failed after 5 attempts"; exit 1
                """
                }
            }
        }
    }
  }