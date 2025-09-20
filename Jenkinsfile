pipeline {
    agent any

    tools {
        jdk 'jdk11'
        maven 'maven3'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Sivakrishnachekuri/Boardgame-CICD.git'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=BoardGame-cicd -Dsonar.projectName=BoardGame-cicd'
                }
            }
        }

        stage('Check Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build & Deploy') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settingsmaven', jdk: 'jdk11', traceability: true) {
                    sh "mvn clean package deploy"
                }
            }
        }

        stage('Docker Image Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cicd2', toolName: 'docker') {
                        sh "docker build -t sivakrishna72/boardgame-cicd:2.0.0 ."
                        sh "docker push sivakrishna72/boardgame-cicd:2.0.0"
                    }
                }
            }
        }

        stage('Sscan docker image') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html sivakrishna72/boardgame-cicd:2.0.0'
            }
        }

        stage('Deploy to Kubernetes and verify the status') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8s-cred',
                    namespace: 'boardgame',
                    serverUrl: 'https://172.31.36.136:6443'
                ) {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl get pods -n boardgame"
                    sh "kubectl get svc -n boardgame"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                // Read Trivy reports
                def fsReport = readFile('trivy-fs-report.html')
                def imageReport = readFile('trivy-image-report.html')
                def fsVulnFound = fsReport.contains('VULNERABILITY')
                def imageVulnFound = imageReport.contains('VULNERABILITY')

                def vulnMessage = ""
                if(fsVulnFound || imageVulnFound) {
                    vulnMessage = "<p style='color:red;'><b>⚠️ Vulnerabilities found in Trivy scan!</b></p>"
                } else {
                    vulnMessage = "<p style='color:green;'><b>✅ No vulnerabilities found in Trivy scans.</b></p>"
                }

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                        <h2>${jobName} - Build ${buildNumber}</h2>
                        <div style="background-color: ${bannerColor}; padding: 10px;">
                            <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                        </div>
                        ${vulnMessage}
                        <p>Check the <a href='${BUILD_URL}'>console output</a>.</p>
                        <p>Attached Trivy Reports:</p>
                        <ul>
                            <li>File System Scan: <b>trivy-fs-report.html</b></li>
                            <li>Docker Image Scan: <b>trivy-image-report.html</b></li>
                        </ul>
                    </div>
                    </body>
                    </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'xyz@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html, trivy-image-report.html'
                )
            }
        }
    }
}
