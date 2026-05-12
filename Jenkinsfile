pipeline {

    agent any

    environment {

        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
        IMAGE_NAME   = "sejalkatre/flask-app"
    }

    stages {

        // -----------------------------------
        // Checkout App Repo
        // -----------------------------------
        stage('Checkout App Repo') {

            steps {

                dir('app-repo') {

                    git branch: 'main',
                        url: 'https://github.com/Sejalkatre/app-repo.git',
                        credentialsId: 'Github-creds'
                }
            }
        }

        // -----------------------------------
        // Checkout Infra Repo
        // -----------------------------------
        stage('Checkout Infra Repo') {

            steps {

                dir('infra-repo') {

                    git branch: 'main',
                        url: 'https://github.com/Sejalkatre/infra-repo.git',
                        credentialsId: 'Github-creds'
                }
            }
        }

        // -----------------------------------
        // Terraform Init
        // -----------------------------------
        stage('Terraform Init') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    dir('infra-repo/terraform') {

                        sh '''
                            rm -rf .terraform
                            rm -f .terraform.lock.hcl

                            terraform init -upgrade
                        '''
                    }
                }
            }
        }

        // -----------------------------------
        // Terraform Validate
        // -----------------------------------
        stage('Terraform Validate') {

            steps {

                dir('infra-repo/terraform') {

                    sh '''
                        terraform validate
                    '''
                }
            }
        }

        // -----------------------------------
        // Terraform Plan
        // -----------------------------------
        stage('Terraform Plan') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    dir('infra-repo/terraform') {

                        sh '''
                            terraform plan -out=tfplan
                        '''
                    }
                }
            }
        }

        // -----------------------------------
        // Terraform Apply
        // -----------------------------------
        stage('Terraform Apply') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    dir('infra-repo/terraform') {

                        sh '''
                            terraform apply -auto-approve tfplan
                        '''
                    }
                }
            }
        }

        // -----------------------------------
        // Configure kubeconfig
        // -----------------------------------
        stage('Configure kubeconfig') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        aws eks update-kubeconfig \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME
                    '''
                }
            }
        }

        // -----------------------------------
        // Install ArgoCD
        // -----------------------------------
        stage('Install ArgoCD') {

            steps {

                sh '''
                    kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -n argocd -f \
                    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
                '''
            }
        }

        // -----------------------------------
        // Build Docker Image
        // -----------------------------------
        stage('Build Docker Image') {

            steps {

                dir('app-repo') {

                    script {

                        docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                    }
                }
            }
        }

        // -----------------------------------
        // Push Docker Image
        // -----------------------------------
        stage('Push Docker Image') {

            steps {

                script {

                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'dockerhub-creds'
                    ) {

                        def appImage = docker.image("${IMAGE_NAME}:${BUILD_NUMBER}")

                        appImage.push("${BUILD_NUMBER}")

                        appImage.push("latest")
                    }
                }
            }
        }

        // -----------------------------------
        // Deploy Application
        // -----------------------------------
        stage('Deploy App') {

            steps {

                dir('app-repo') {

                    sh '''
                        kubectl apply -f k8s/
                    '''
                }
            }
        }

        // -----------------------------------
        // ArgoCD Sync
        // -----------------------------------
        stage('ArgoCD Sync') {

            steps {

                sh '''
                    argocd app sync flask-app || true
                '''
            }
        }
    }

    // -----------------------------------
    // Post Actions
    // -----------------------------------
    post {

        success {

            echo 'Pipeline completed successfully.'
        }

        failure {

            echo 'Pipeline failed.'
        }
    }
}
