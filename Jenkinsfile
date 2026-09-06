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
                    container('jnlp') {
                        // יצירת תבנית HTML מקומית ישירות ב-Workspace כדי למנוע שגיאות 404
                        sh '''cat << 'EOF' > html.tpl
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Trivy Vulnerability Report</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background-color: #f4f6f9; }
    h1 { color: #333; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; background: #fff; }
    th, td { padding: 10px; border: 1px solid #ddd; text-align: left; }
    th { background-color: #343a40; color: white; }
    .HIGH { background-color: #ffc107; font-weight: bold; }
    .CRITICAL { background-color: #dc3545; color: white; font-weight: bold; }
  </style>
</head>
<body>
  <h1>Trivy Vulnerability Report</h1>
  <table>
    <tr><th>Target</th><th>Library</th><th>Vulnerability</th><th>Severity</th><th>Installed</th><th>Fixed Version</th></tr>
    {{ range . }}
      {{ range .Vulnerabilities }}
      <tr>
        <td>{{ $.Target }}</td>
        <td>{{ .PkgName }}</td>
        <td><a href="{{ .PrimaryURL }}" target="_blank">{{ .VulnerabilityID }}</a></td>
        <td class="{{ .Severity }}">{{ .Severity }}</td>
        <td>{{ .InstalledVersion }}</td>
        <td>{{ .FixedVersion }}</td>
      </tr>
      {{ end }}
    {{ end }}
  </table>
</body>
</html>
EOF
'''
                    }

                    // יצירת דוח ה-HTML בעזרת התבנית המקומית
                    container('trivy') {
                        sh "trivy image --format template --template '@html.tpl' --output trivy-report.html ${appimage}:${apptag}"
                    }

                    // שמירת הדוח כ-Artifact לצפייה ושימוש ב-Jenkins
                    archiveArtifacts artifacts: 'trivy-report.html', allowEmptyArchive: true

                    // הכשלת ה-Pipeline אם קיימות חולשות HIGH/CRITICAL
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
