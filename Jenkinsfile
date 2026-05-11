pipeline {
    agent any

    stages {

        stage('Checkout App Repo') {
            steps {

                dir('app-repo') {

                    git branch: 'main',
                        url: 'https://github.com/Sejalkatre/app-repo.git',
                        credentialsId: 'Github-creds'
                }
            }
        }

        stage('Checkout Infra Repo') {
            steps {

                dir('infra-repo') {

                    git branch: 'main',
                        url: 'https://github.com/Sejalkatre/infra-repo.git',
                        credentialsId: 'Github-creds'
                }
            }
        }

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

        stage('Terraform Apply (Manual Approval)') {
            steps {

                script {

                    timeout(time: 10, unit: 'MINUTES') {

                        input message: "Do you want to apply Terraform changes?"
                    }

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
        }

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
                        --region us-west-2 \
                        --name devops-cluster
                    '''
                }
            }
        }

        stage('Install ArgoCD') {
            steps {

                sh '''
                    kubectl create namespace argocd || true

                    kubectl apply -n argocd -f \
                    https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
                '''
            }
        }

        stage('Build Docker Image') {
            steps {

                script {

                    docker.build(
                        "sejalkatre/flask-app:${env.BUILD_NUMBER}"
                    )
                }
            }
        }

        stage('Push to DockerHub') {
            steps {

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

        stage('ArgoCD Sync') {
            steps {

                sh '''
                    argocd app sync flask-app || true
                '''
            }
        }
    }

    post {

        failure {

            echo "Pipeline failed. Destroying Terraform infrastructure..."

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

        success {

            echo "Pipeline completed successfully."
        }
    }
}
