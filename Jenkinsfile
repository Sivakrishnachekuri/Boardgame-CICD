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
                    // Do not abort the pipeline, just check status
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

        stage('Docker Image Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cicd2', toolName: 'docker') {
                        sh "docker build -t sivakrishna72/boardgame-cicd:1.0.0 ."
                        sh "docker push sivakrishna72/boardgame-cicd:1.0.0 "
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o trivy-image-report.html sivakrishna72/boardgame-cicd:1.0.0'
            }
        }
    }

    post {
        success {
            echo '✅ Build, scan, QA, and deploy completed successfully!'
        }
        failure {
            echo '❌ Something failed. Check logs.'
        }
    }
}
