def appname = "hello-newapp"
def repo = "hillel456" // שם המשתמש שלך
def appimage = "${repo}/${appname}"
def apptag = "${env.BUILD_NUMBER}"

podTemplate(containers: [
    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
    containerTemplate(
        name: 'docker', 
        image: 'docker:26-dind',
        privileged: true,
        args: '--storage-driver=vfs --host=tcp://0.0.0.0:2375'
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

        stage('Build & Push') {
            container('docker') {
                echo "Building docker image..."
                sh "docker build . -t ${appimage}:${apptag} -t ${appimage}:latest"
                
                echo "Pushing to DockerHub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${appimage}:${apptag}"
                    sh "docker push ${appimage}:latest"
                }
            }
        }
    }
}
