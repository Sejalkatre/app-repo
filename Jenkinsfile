pipeline {
    agent any

    environment {
        // Use Jenkins credentials IDs
        AWS_CREDS = credentials('aws-creds')          // AWS Access Key/Secret
        DOCKERHUB_CREDS = credentials('dockerhub-creds') // DockerHub username/password
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Init') {
            steps {
                dir('../infra-repo/terraform') {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                        export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW
                        terraform init
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir('../infra-repo/terraform') {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                        export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW
                        terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Terraform Apply (Manual Approval)') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        input message: "Do you want to apply Terraform changes?"
                    }
                    dir('../infra-repo/terraform') {
                        sh '''
                            export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                            export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW
                            terraform apply tfplan
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def app = docker.build("sejalkatre/flask-app:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("sejalkatre/flask-app:${env.BUILD_NUMBER}")
                        app.push()
                        app.push("latest") // also push latest tag
                    }
                }
            }
        }

        stage('ArgoCD Sync') {
            steps {
                script {
                    // Optional: ArgoCD auto-syncs anyway
                    sh 'argocd app sync flask-app || true'
                }
            }
        }
    }
}
