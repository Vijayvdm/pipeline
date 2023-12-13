pipeline {
    agent any
    environment {
    SLACK_CHANNEL = 'jenkins-testing'
    }

    tools {
        maven "Maven"
    }

    stages {
        stage('checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Vijayvdm/docker.git'
            }
        }
        
        stage('Execute Maven') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Docker Build and Tag') {
            steps {
                // Build and tag Docker image 
                sh 'docker build -t samplewebapp:latest .'
                sh 'docker tag samplewebapp:latest aswarda/samplewebapp:latest'
            }
        }

        stage('Publish image to Docker Hub') {
            steps {
                // Push Docker image to Docker Hub 2nd container#
                script {
                    withDockerRegistry([credentialsId: "dockerHub", url: ""]) {
                        sh 'docker push aswarda/samplewebapp:latest'
                    }
                }
            }
        }

        stage('Run Docker container on Jenkins Agent') {
            steps {
                // Run Docker container on Jenkins Agent
                sh 'docker run -d -p 8015:8080 aswarda/samplewebapp:latest'
            }
        }

        stage('Run Docker container on remote hosts') {
            steps {
                // Copy Docker image to remote host and run container
                script {
                    sh "docker save aswarda/samplewebapp:latest | docker load"
                    sh "docker run -d -p 8014:8080 aswarda/samplewebapp:latest"
                }
            }
        }
    }
    
    post {
        success {
            slackSend(color: '#00FF00', message: "Job '${env.JOB_NAME} ${env.BUILD_NUMBER}' has succeeded!", channel: SLACK_CHANNEL)
        }
        failure {
            slackSend(color: '#FF0000', message: "Job '${env.JOB_NAME} ${env.BUILD_NUMBER}' has failed. Check the console output for more information.", channel: SLACK_CHANNEL)
        }
    }
}
