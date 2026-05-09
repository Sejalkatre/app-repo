pipeline {
  agent any
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build Docker Image') {
      steps { sh 'docker build -t flask-app:latest .' }
    }
    stage('Push to Registry') {
      steps {
        sh 'docker tag flask-app:latest Sejalkatre/flask-app:latest'
        sh 'docker push Sejalkatre/flask-app:latest'
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        sh 'kubectl apply -f ../infra-repo/manifests/deployment.yaml'
        sh 'kubectl apply -f ../infra-repo/manifests/service.yaml'
        sh 'kubectl apply -f ../infra-repo/manifests/hpa.yaml'
      }
    }
  }
}
