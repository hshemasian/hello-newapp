def appname = "hello-newapp"
def repo = "hillel456"
def appimage = "${repo}/${appname}"
def apptag = "${env.BUILD_NUMBER}"

podTemplate(containers: [
    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
    containerTemplate(
        name: 'docker', 
        image: 'docker:26-dind',
        privileged: true,
        envVars: [
            envVar(key: 'DOCKER_TLS_CERTDIR', value: '')
        ],
        args: '--storage-driver=vfs'
    ),
    containerTemplate(
        name: 'trivy', 
        image: 'aquasec/trivy:latest',
        ttyEnabled: true,
        command: 'cat'
    )
  ],
  volumes: [
    emptyDirVolume(mountPath: '/var/run', memory: false) 
  ]) 
  {
    node(POD_LABEL) {
        stage('Checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('Build') {
            container('docker') {
                sh "docker build . -t ${appimage}:${apptag} -t ${appimage}:latest"
            }
        }

        stage('Parallel Tasks') {
            parallel(
                "Task 1": {
                    container('jnlp') {
                        sh "echo 'Running parallel checks...'"
                    }
                },
                "Task 2 - Trivy Scan": {
                    container('trivy') {
                        sh "trivy image --severity HIGH,CRITICAL ${appimage}:${apptag}"
                    }
                }
            )
        }

        stage('Push to DockerHub') {
            container('docker') {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${appimage}:${apptag}"
                    sh "docker push ${appimage}:latest"
                }
            }
        }
    }
}
