pipeline {

    agent any

    environment {

        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
        IMAGE_NAME   = "sejalkatre/flask-app"

        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    options {

        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        // =====================================================
        // Clean Workspace
        // =====================================================

        stage('Clean Workspace') {

            steps {

                cleanWs()
            }
        }

        // =====================================================
        // Checkout App Repo
        // =====================================================

        stage('Checkout App Repo') {

            steps {

                dir('app-repo') {

                    git branch: 'main',
                        credentialsId: 'Github-creds',
                        url: 'https://github.com/Sejalkatre/app-repo.git'
                }
            }
        }

        // =====================================================
        // Checkout Infra Repo
        // =====================================================

        stage('Checkout Infra Repo') {

            steps {

                dir('infra-repo') {

                    git branch: 'main',
                        credentialsId: 'Github-creds',
                        url: 'https://github.com/Sejalkatre/infra-repo.git'
                }
            }
        }

        // =====================================================
        // Terraform Init
        // =====================================================

        stage('Terraform Init') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

                            export AWS_DEFAULT_REGION=$AWS_REGION

                            terraform init

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Terraform Validate
        // =====================================================

        stage('Terraform Validate') {

            steps {

                dir('infra-repo/terraform') {

                    sh '''

                        terraform validate

                    '''
                }
            }
        }

        // =====================================================
        // Terraform Plan
        // =====================================================

        stage('Terraform Plan') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

                            export AWS_DEFAULT_REGION=$AWS_REGION

                            terraform plan -out=tfplan

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Terraform Apply
        // =====================================================

        stage('Terraform Apply') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

                            export AWS_DEFAULT_REGION=$AWS_REGION

                            terraform apply -auto-approve tfplan

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Wait For EKS
        // =====================================================

        stage('Wait For EKS') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''

                        export AWS_DEFAULT_REGION=$AWS_REGION

                        aws eks wait cluster-active \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME

                    '''
                }
            }
        }

        // =====================================================
        // Configure kubeconfig
        // =====================================================

        stage('Configure kubeconfig') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''

                        export AWS_DEFAULT_REGION=$AWS_REGION

                        mkdir -p ~/.kube

                        aws eks update-kubeconfig \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME

                        kubectl get nodes

                    '''
                }
            }
        }

        // =====================================================
        // Install ArgoCD
        // =====================================================

        stage('Install ArgoCD') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''

                        export AWS_DEFAULT_REGION=$AWS_REGION

                        kubectl create namespace argocd \
                        --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply \
                        --server-side \
                        -n argocd \
                        -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

                        kubectl rollout status deployment/argocd-server \
                        -n argocd \
                        --timeout=300s

                    '''
                }
            }
        }

        // =====================================================
        // Apply ArgoCD Application
        // =====================================================

        stage('Apply ArgoCD Application') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/argocd') {

                        sh '''

                            kubectl apply -f application.yaml

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Build Docker Image
        // =====================================================

        stage('Build Docker Image') {

            steps {

                dir('app-repo') {

                    script {

                        def appImage = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")

                    }
                }
            }
        }

        // =====================================================
        // Push Docker Image
        // =====================================================

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

        // =====================================================
        // Verify Deployment
        // =====================================================

        stage('Verify Deployment') {

            steps {

                sh '''

                    kubectl get nodes

                    kubectl get pods -A

                    kubectl get svc -A

                    kubectl get ingress -A

                '''
            }
        }
    }

    // =====================================================
    // Post Actions
    // =====================================================

    post {

        success {

            echo 'Pipeline completed successfully.'
        }

        failure {

            echo 'Pipeline failed.'
        }
    }
}
