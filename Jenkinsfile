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
        // INIT AWS + KUBE ACCESS ONCE (IMPORTANT FIX)
        // =====================================================
        stage('Setup AWS & Kubeconfig') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''
                        export AWS_DEFAULT_REGION=$AWS_REGION

                        aws sts get-caller-identity

                        mkdir -p ~/.kube

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $CLUSTER_NAME

                        kubectl get nodes
                    '''
                }
            }
        }

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

        stage('Terraform Validate') {
            steps {
                dir('infra-repo/terraform') {
                    sh 'terraform validate'
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
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
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
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

        stage('Wait For EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {

                    sh '''
                        aws eks wait cluster-active \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME
                    '''
                }
            }
        }

        stage('Install ArgoCD') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {

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

            withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds'
            ]]) {

                dir('infra-repo/terraform') {
                    sh '''
                        terraform destroy -auto-approve || true
                    '''
                }
            }
        }
    }
}
