pipeline {

    agent any

    environment {

        INFRA_CHANGED = "false"

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
        // Detect Terraform Changes
        // -----------------------------------
        stage('Check Infra Changes') {

            steps {

                script {

                    def infraChanges = sh(
                        script: '''
                            cd infra-repo

                            git diff --name-only HEAD~1 HEAD | grep terraform || true
                        ''',
                        returnStdout: true
                    ).trim()

                    if (infraChanges) {

                        env.INFRA_CHANGED = "true"

                        echo "Terraform changes detected."

                    } else {

                        env.INFRA_CHANGED = "false"

                        echo "No Terraform changes detected."
                    }
                }
            }
        }

        // -----------------------------------
        // Terraform Init
        // -----------------------------------
        stage('Terraform Init') {

            when {
                expression {
                    env.INFRA_CHANGED == "true"
                }
            }

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
        // Terraform Plan
        // -----------------------------------
        stage('Terraform Plan') {

            when {
                expression {
                    env.INFRA_CHANGED == "true"
                }
            }

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

            when {
                expression {
                    env.INFRA_CHANGED == "true"
                }
            }

            steps {

                timeout(time: 10, unit: 'MINUTES') {

                    input message: 'Apply Terraform Changes?'
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

        // -----------------------------------
        // Configure kubeconfig
        // -----------------------------------
        stage('Configure kubeconfig') {

            when {
                expression {
                    env.INFRA_CHANGED == "true"
                }
            }

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

            when {
                expression {
                    env.INFRA_CHANGED == "true"
                }
            }

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

        failure {

            script {

                if (env.INFRA_CHANGED == "true") {

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
            }
        }

        success {

            echo "Pipeline completed successfully."
        }
    }
}
