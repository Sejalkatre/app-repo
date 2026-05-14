pipeline {

    agent any

    environment {

        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
        IMAGE_NAME   = "sejalkatre/flask-app"
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
        // Detect Changes
        // =====================================================

        stage('Detect Changes') {

            steps {

                script {

                    env.INFRA_CHANGED = sh(
                        script: '''
                            cd infra-repo
                            git diff --name-only HEAD~1 HEAD | wc -l
                        ''',
                        returnStdout: true
                    ).trim()

                    env.APP_CHANGED = sh(
                        script: '''
                            cd app-repo
                            git diff --name-only HEAD~1 HEAD | wc -l
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "INFRA_CHANGED = ${env.INFRA_CHANGED}"

                    echo "APP_CHANGED = ${env.APP_CHANGED}"
                }
            }
        }

        // =====================================================
        // Terraform Init
        // =====================================================

        stage('Terraform Init') {

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

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

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

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

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

                            terraform plan -out=tfplan

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Manual Approval
        // =====================================================

        stage('Manual Approval') {

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                timeout(time: 10, unit: 'MINUTES') {

                    input message: 'Approve Terraform Apply?',
                          ok: 'Apply'
                }
            }
        }

        // =====================================================
        // Terraform Apply
        // =====================================================

        stage('Terraform Apply') {

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/terraform') {

                        sh '''

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

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                sleep(time: 120, unit: 'SECONDS')
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

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''

                        kubectl create namespace argocd \
                        --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply -n argocd -f \
                        https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

                        kubectl rollout status deployment/argocd-server \
                        -n argocd --timeout=300s

                    '''
                }
            }
        }

        // =====================================================
        // Apply ArgoCD Application
        // =====================================================

        stage('Apply ArgoCD Application') {

            when {

                expression {

                    env.INFRA_CHANGED != "0"
                }
            }

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

            when {

                expression {

                    env.APP_CHANGED != "0"
                }
            }

            steps {

                dir('app-repo') {

                    script {

                        docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
                    }
                }
            }
        }

        // =====================================================
        // Push Docker Image
        // =====================================================

        stage('Push Docker Image') {

            when {

                expression {

                    env.APP_CHANGED != "0"
                }
            }

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
        // Deploy Kubernetes Manifests
        // =====================================================

        stage('Deploy Application') {

            when {

                expression {

                    env.APP_CHANGED != "0" || env.INFRA_CHANGED != "0"
                }
            }

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    dir('infra-repo/manifests') {

                        sh '''

                            kubectl apply -f .

                        '''
                    }
                }
            }
        }

        // =====================================================
        // Verify Deployment
        // =====================================================

        stage('Verify Deployment') {

            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {

                    sh '''

                        kubectl get nodes

                        kubectl get pods -A

                        kubectl get svc -A

                        kubectl get ingress -A

                    '''
                }
            }
        }
    }

    post {

        success {

            echo 'Pipeline completed successfully.'
        }

        failure {

            echo 'Pipeline failed.'

            script {

                if (env.INFRA_CHANGED != "0") {

                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
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
    }
}
