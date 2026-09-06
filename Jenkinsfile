def appname = "hello-newapp"
def repo = "hillel456"
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
        stage('chackout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('Hello') {
            container('docker') {
                echo "Building docker image..."
                sh "echo docker push $appimage"
            }
        }
    }
}
