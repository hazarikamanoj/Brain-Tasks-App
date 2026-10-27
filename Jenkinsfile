pipeline {
  agent any
  environment {
    AWS_REGION  = 'us-east-1'
    ECR_REPO    = 'brain-tasks-app'
    EKS_CLUSTER = 'brain-tasks-cluster'
    ACCOUNT_ID  = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
    REGISTRY    = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    REPO_URI    = "${REGISTRY}/${ECR_REPO}"
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script { env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim() }
      }
    }
    stage('Build Image') {
      steps { sh 'docker build -t $REPO_URI:$IMAGE_TAG -t $REPO_URI:latest .' }
    }
    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REGISTRY
          docker push $REPO_URI:$IMAGE_TAG
          docker push $REPO_URI:latest
        '''
      }
    }
    stage('Deploy to EKS') {
      steps {
        sh '''
          aws eks update-kubeconfig --name $EKS_CLUSTER --region $AWS_REGION
          sed -i "s|image:.*|image: $REPO_URI:$IMAGE_TAG|" k8s/deployment.yaml
          kubectl apply -f k8s/
          kubectl rollout status deployment/brain-tasks-app --timeout=180s
        '''
      }
    }
  }
}