pipeline {
    agent any

    environment {
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        IMAGE_NAME = 'nginx-cicd'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:build-${BUILD_NUMBER} .'
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker tag ${IMAGE_NAME}:build-${BUILD_NUMBER} $DOCKERHUB_USER/${IMAGE_NAME}:build-${BUILD_NUMBER}
                        docker push $DOCKERHUB_USER/${IMAGE_NAME}:build-${BUILD_NUMBER}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml
                        kubectl set image deployment/nginx-app nginx=$DOCKERHUB_USER/$IMAGE_NAME:build-${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Rolling Update') {
            steps {
                sh 'kubectl rollout status deployment/nginx-app --timeout=120s'
            }
        }
    }
}
