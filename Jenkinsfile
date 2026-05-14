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

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout App Repo') {
            steps {
                dir('app-repo') {
                    git branch: 'main',
                        credentialsId: 'Github-creds',
                        url: 'https://github.com/Sejalkatre/app-repo.git'
                }
            }
        }

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
        // TERRAFORM (CREATE EKS FIRST)
        // =====================================================

        stage('Terraform Init') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
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

        stage('Terraform Validate') {
            steps {
                dir('infra-repo/terraform') {
                    sh 'terraform validate'
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
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

        stage('Terraform Apply') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
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
        // WAIT FOR EKS
        // =====================================================

        stage('Wait For EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
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
        // CONFIGURE KUBECONFIG (IMPORTANT FIX)
        // =====================================================

        stage('Configure kubeconfig') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {

                    sh '''
                        export AWS_DEFAULT_REGION=$AWS_REGION

                        mkdir -p ~/.kube

                        aws eks list-clusters --region $AWS_REGION
                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $CLUSTER_NAME

                        kubectl get nodes
                    '''
                }
            }
        }

        // =====================================================
        // ARGOCD INSTALL
        // =====================================================

        stage('Install ArgoCD') {
            steps {
                sh '''
                    kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -n argocd -f \
                    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

                    sleep 60

                    kubectl wait --for=condition=available deployment/argocd-server \
                    -n argocd --timeout=600s
                '''
            }
        }

        stage('Apply ArgoCD Application') {
            steps {
                dir('infra-repo/argocd') {
                    sh '''
                        kubectl apply -f application.yaml
                    '''
                }
            }
        }

        // =====================================================
        // DOCKER BUILD & PUSH
        // =====================================================

        stage('Build Docker Image') {
            steps {
                dir('app-repo') {
                    script {
                        docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                    }
                }
            }
        }

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
        // VERIFY DEPLOYMENT
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

    post {

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'

            dir('infra-repo/terraform') {
                sh '''
                    terraform init
                    terraform destroy -auto-approve || true
                '''
            }
        }
    }
}
