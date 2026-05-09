pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Terraform Init') {
      steps {
        dir('../infra-repo/terraform') {
          sh 'terraform init'
        }
      }
    }

    stage('Terraform Plan') {
      steps {
        dir('../infra-repo/terraform') {
          sh 'terraform plan -out=tfplan'
        }
      }
    }

    stage('Terraform Apply (Manual Approval)') {
      steps {
        script {
          timeout(time: 10, unit: 'MINUTES') {
            input message: "Do you want to apply Terraform changes?"
          }
          dir('../infra-repo/terraform') {
            sh 'terraform apply tfplan'
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t flask-app:latest .'
      }
    }

    stage('Push to Registry') {
      steps {
        sh 'docker tag flask-app:latest Sejalkatre/flask-app:latest'
        sh 'docker push Sejalkatre/flask-app:latest'
      }
    }

    stage('ArgoCD Sync') {
      steps {
        script {
          // Optional: ArgoCD auto-syncs anyway
          sh 'argocd app sync flask-app'
        }
      }
    }
  }
}
