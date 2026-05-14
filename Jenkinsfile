pipeline {

    agent any

    environment {

        AWS_REGION   = "us-west-2"
        CLUSTER_NAME = "devops-cluster"
        IMAGE_NAME   = "sejalkatre/flask-app"
    }

    stages {

        // =====================================================
        // Checkout App Repo
        // =====================================================
        stage('Checkout App Repo') {

            steps {

                dir('app-repo') {

                    git branch: 'main',
                        url: 'https://github.com/Sejalkatre/app-repo.git',
                        credentialsId: 'Github-creds'
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
                        url: 'https://github.com/Sejalkatre/infra-repo.git',
                        credentialsId: 'Github-creds'
                }
            }
        }

        // =====================================================
        // Terraform Init
        // =====================================================
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

        // =====================================================
        // Terraform Apply
        // =====================================================
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

        // =====================================================
        // Configure kubeconfig
        // =====================================================
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

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    dir('infra-repo/argocd') {

                        sh '''
                            kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

                            kubectl apply -f .
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

                        docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")
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
        // Deploy Application
        // =====================================================
        stage('Deploy Application') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-creds',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

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

                sh '''
                    kubectl get nodes
                    kubectl get pods -A
                    kubectl get svc -A
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

        always {

            script {

                if (currentBuild.currentResult == 'FAILURE') {

                    echo 'Build failed. Destroying infrastructure...'

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
                                terraform destroy -auto-approve || true
                            '''
                        }
                    }
                }
            }
        }
    }
}
