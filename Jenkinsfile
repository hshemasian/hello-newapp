def appname = "hello-newapp"
def repo = "hillel456" // שם המשתמש שלך ב-DockerHub
def appimage = "docker.io/${repo}/${appname}"
def apptag = "${env.BUILD_NUMBER}"

podTemplate(cloud: 'kubernetes', containers: [
    containerTemplate(
        name: 'jnlp', 
        image: 'jenkins/inbound-agent:latest'
    ),
    containerTemplate(
        name: 'docker', 
        image: 'docker:26-dind',
        privileged: true,
        args: '--storage-driver=vfs'
    )], 
  volumes: [
    emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
  ]) {
    node(POD_LABEL) {
        stage('Checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('Build & Push') {
            container('docker') {
                echo "Building Docker image..."
                sh "docker build -t ${appimage}:${apptag} -t ${appimage}:latest ."

                echo "Pushing Docker image to DockerHub..."
                // התחברות ודחיפה באמצעות Credentials מ-Jenkins
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${appimage}:${apptag}"
                    sh "docker push ${appimage}:latest"
                }
            }
        }
    }
}
