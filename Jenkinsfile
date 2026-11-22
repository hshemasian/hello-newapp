// GitOps flow: build the image (docker-in-docker), push tags <BUILD_NUMBER> + latest,
// render the Helm chart, and commit the rendered manifests to
//   https://github.com/elevy99927-demos/gitops  ->  systems/<DEPARTMENT>/<ENV>/<user>.yaml
// Argo CD (ApplicationSet/systems-manifests.yaml) deploys that file.



def user        = "hillel456"               // your name: Docker Hub user AND gitops file name
def appname     = "RecruitHub"
def appimage    = "${user}/${appname.toLowerCase()}"      // docker.io/<user>/recruithub
def apptag      = "${env.BUILD_NUMBER}"
def envName     = "dev"
def department  = "hr"

def gitopsRepo  = "github.com/elevy99927-demos/gitops.git"
def gitopsFile  = "systems/${department}/${envName}/${user}.yaml"

podTemplate(containers: [
      containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
      containerTemplate(name: 'deployer', image: 'elevy99927/k8s-deployer:latest', command: 'cat', ttyEnabled: true),
      // docker-in-docker: this container runs dockerd; the sh steps below run the docker CLI in it
      containerTemplate(name: 'docker', image: 'docker:27-dind', privileged: true,
                        args: '--storage-driver=vfs',
                        envVars: [envVar(key: 'DOCKER_TLS_CERTDIR', value: '')])   // no TLS inside the pod
  ])  {
    // Credentials come from Jenkins (Manage Jenkins -> Credentials), not from Kubernetes secrets:
    //   docker-creds       : Secret text - Docker Hub access token
    //   github-credentials : Username with password - hshemasian + GitHub token (write access to gitops)
    node(POD_LABEL) {
        stage('checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('build') {
            container('docker') {
                // wait for dockerd (started by the dind container) to accept connections
                sh '''
                    for i in $(seq 1 30); do docker info >/dev/null 2>&1 && exit 0; sleep 2; done
                    echo "docker daemon did not start"; docker info; exit 1
                '''
                echo "Building ${appimage}:${apptag}"
                // quote the workspace: job names with spaces give paths with spaces
                sh "docker build -t ${appimage}:${apptag} -t ${appimage}:latest \"${env.WORKSPACE}\""
            }
        }

        stage('push') {
            container('docker') {
                // TAGS: <BUILD_NUMBER> (immutable, deployed by GitOps) and latest
                withCredentials([string(credentialsId: 'docker-creds', variable: 'DOCKER_TOKEN')]) {
                    sh """
                        set -e
                        echo "\$DOCKER_TOKEN" | docker login -u "${user}" --password-stdin
                        docker push ${appimage}:${apptag}
                        docker push ${appimage}:latest
                        docker logout
                    """
                }
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
                    git config user.name  "Jenkins (${user})"
                    git config user.email "elevy99927-demos@users.noreply.github.com"

                    helm template ${user} "${env.WORKSPACE}/chart" \\
                        --set image.repository=${appimage} \\
                        --set image.tag=${apptag} \\
                        > ${gitopsFile}

                    git add ${gitopsFile}
                    if git diff --cached --quiet; then
                        echo "No changes in ${gitopsFile} - nothing to deploy"
                        exit 0
                    fi
                    git commit -m "Deploy ${appimage}:${apptag} to ${department}/${envName} (build ${env.BUILD_NUMBER}) \n Commited by ${user}"

                    git push origin HEAD:main
                """
                }
            }
        }

        stage('add-dashboard') {
            container('deployer') {
                sh """
                    echo " TBD: Add dashboard to grafana"
                """
            }
        }
    }
  }