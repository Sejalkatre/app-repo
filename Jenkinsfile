pipeline {

    agent any

    environment {

        AWS_REGION = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
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
                            terraform init
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
                    kubectl create namespace argocd || true

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

                        docker.build(
                            "sejalkatre/flask-app:${env.BUILD_NUMBER}"
                        )
                    }
                }
            }
        }

        // -----------------------------------
        // Push Docker Image
        // -----------------------------------
        stage('Push Docker Image') {

            steps {

                dir('app-repo') {

                    script {

                        docker.withRegistry(
                            'https://index.docker.io/v1/',
                            'dockerhub-creds'
                        ) {

                            def app = docker.build(
                                "sejalkatre/flask-app:${env.BUILD_NUMBER}"
                            )

                            app.push()

                            app.push("latest")
                        }
                    }
                }
            }
        }

        // -----------------------------------
        // Verify Cluster
        // -----------------------------------
        stage('Verify Cluster') {

            steps {

                sh '''
                    kubectl get nodes
                    kubectl get pods -A
                '''
            }
        }
    }

    post {

        success {

            echo "Pipeline completed successfully."
        }

        failure {

            echo "Pipeline failed."
        }
    }
}
