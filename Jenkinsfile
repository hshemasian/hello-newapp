def appname = "hello-newapp"
def repo = "hillel456"
def appimage = "${repo}/${appname}"

podTemplate(containers: [
    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent', ttyEnabled: true),
    containerTemplate(
        name: 'docker', 
        image: 'docker:26-dind',
        privileged: true,
        envVars: [
            envVar(key: 'DOCKER_TLS_CERTDIR', value: ''),
            envVar(key: 'DOCKER_HOST', value: 'tcp://localhost:2375')
        ],
        args: '--storage-driver=vfs'
    ),
    containerTemplate(
        name: 'trivy', 
        image: 'aquasec/trivy:latest',
        ttyEnabled: true,
        command: 'cat',
        envVars: [
            envVar(key: 'DOCKER_HOST', value: 'tcp://localhost:2375')
        ]
    )
  ]
) {
    node(POD_LABEL) {
        def apptag = "${env.BUILD_NUMBER}"

        stage('Checkout') {
            container('jnlp') {
                sh '/usr/bin/git config --global http.sslVerify false'
                checkout scm
            }
        }

        stage('Build') {
            container('docker') {
                sh """
                    until docker info > /dev/null 2>&1; do
                        echo "Waiting for Docker daemon..."
                        sleep 1
                    done
                    docker build . -t ${appimage}:${apptag} -t ${appimage}:latest
                """
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
                    // 1. הורדת תבנית ה-HTML דרך קונטיינר jnlp (שכולל curl) לתוך ה-Workspace
                    container('jnlp') {
                        sh 'curl -sSL https://raw.githubusercontent.com/aquasec/trivy/main/contrib/html.tpl -o html.tpl'
                    }

                    // 2. יצירת דוח HTML מעוצב
                    container('trivy') {
                        sh "trivy image --format template --template '@html.tpl' --output trivy-report.html ${appimage}:${apptag}"
                    }

                    // 3. שמירת דוח ה-HTML כ-Artifact ב-Jenkins לצפייה בדפדפן
                    archiveArtifacts artifacts: 'trivy-report.html', allowEmptyArchive: true

                    // 4. הכשלת ה-Pipeline במידה ונמצאו חולשות ברמת HIGH או CRITICAL
                    container('trivy') {
                        sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${appimage}:${apptag}"
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
