pipeline {

    agent any

    environment {

        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
        IMAGE_NAME   = "sejalkatre/flask-app"

        INFRA_CHANGED = "false"
        APP_CHANGED   = "false"
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

                    INFRA_CHANGED = sh(
                        script: """
                            cd infra-repo
                            git diff --name-only HEAD~1 HEAD | wc -l
                        """,
                        returnStdout: true
                    ).trim()

                    APP_CHANGED = sh(
                        script: """
                            cd app-repo
                            git diff --name-only HEAD~1 HEAD | wc -l
                        """,
                        returnStdout: true
                    ).trim()

                    echo "INFRA_CHANGED = ${INFRA_CHANGED}"

                    echo "APP_CHANGED = ${APP_CHANGED}"
                }
            }
        }

        // =====================================================
        // Terraform Init
        // =====================================================

        stage('Terraform Init') {

            when {

                expression {

                    INFRA_CHANGED != "0"
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

                    INFRA_CHANGED != "0"
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

                    INFRA_CHANGED != "0"
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

                    INFRA_CHANGED != "0"
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

                    INFRA_CHANGED != "0"
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
        // Wait For EKS Cluster
        // =====================================================

        stage('Wait For EKS') {

            when {

                expression {

                    INFRA_CHANGED != "0"
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

                    INFRA_CHANGED != "0"
                }
            }

            steps {

                dir('infra-repo/argocd') {

                    sh '''

                        kubectl create namespace argocd \
                        --dry-run=client -o yaml | kubectl apply -f -

                        kubectl apply -f .

                    '''
                }
            }
        }

        // =====================================================
        // Build Docker Image
        // =====================================================

        stage('Build Docker Image') {

            when {

                expression {

                    APP_CHANGED != "0"
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

                    APP_CHANGED != "0"
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
        // Deploy Application
        // =====================================================

        stage('Deploy Application') {

            when {

                expression {

                    APP_CHANGED != "0" || INFRA_CHANGED != "0"
                }
            }

            steps {

                dir('infra-repo/manifests') {

                    sh '''

                        kubectl apply -f .

                    '''
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

    post {

        success {

            echo 'Pipeline completed successfully.'
        }

        failure {

            echo 'Pipeline failed.'
        }
    }
}
