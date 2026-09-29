pipeline {
    agent any

    environment {
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        IMAGE_APP1 = 'nginx-cicd-app1'
        IMAGE_APP2 = 'nginx-cicd-app2'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD^ HEAD 2>/dev/null || git ls-tree -r --name-only HEAD',
                        returnStdout: true
                    ).trim()

                    def files = changedFiles ? changedFiles.readLines() : []

                    env.BUILD_APP1 = files.any {
                        it.startsWith('app1/')
                    } ? 'true' : 'false'

                    env.BUILD_APP2 = files.any {
                        it.startsWith('app2/')
                    } ? 'true' : 'false'

                    env.DEPLOY_APP1 = (
                        env.BUILD_APP1 == 'true' ||
                        files.contains('app1-deployment.yaml')
                    ) ? 'true' : 'false'

                    env.DEPLOY_APP2 = (
                        env.BUILD_APP2 == 'true' ||
                        files.contains('app2-deployment.yaml')
                    ) ? 'true' : 'false'

                    echo "Build App1: ${env.BUILD_APP1}"
                    echo "Build App2: ${env.BUILD_APP2}"
                    echo "Deploy App1: ${env.DEPLOY_APP1}"
                    echo "Deploy App2: ${env.DEPLOY_APP2}"
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -e

                        echo "$DOCKERHUB_TOKEN" | docker login \
                            -u "$DOCKERHUB_USER" --password-stdin

                        if [ "$BUILD_APP1" = "true" ]; then
                            docker build \
                                -t $DOCKERHUB_USER/$IMAGE_APP1:build-${BUILD_NUMBER} \
                                -t $DOCKERHUB_USER/$IMAGE_APP1:latest \
                                ./app1

                            docker push $DOCKERHUB_USER/$IMAGE_APP1:build-${BUILD_NUMBER}
                            docker push $DOCKERHUB_USER/$IMAGE_APP1:latest
                        fi

                        if [ "$BUILD_APP2" = "true" ]; then
                            docker build \
                                -t $DOCKERHUB_USER/$IMAGE_APP2:build-${BUILD_NUMBER} \
                                -t $DOCKERHUB_USER/$IMAGE_APP2:latest \
                                ./app2

                            docker push $DOCKERHUB_USER/$IMAGE_APP2:build-${BUILD_NUMBER}
                            docker push $DOCKERHUB_USER/$IMAGE_APP2:latest
                        fi

                        docker logout || true
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
                        set -e

                        kubectl apply -f app1-service.yaml
                        kubectl apply -f app2-service.yaml
                        kubectl apply -f ingress.yaml

                        if [ "$DEPLOY_APP1" = "true" ]; then

                            if [ "$BUILD_APP1" = "true" ]; then
                                APP1_IMAGE="$DOCKERHUB_USER/$IMAGE_APP1:build-${BUILD_NUMBER}"
                            else
                                APP1_IMAGE=$(kubectl get deployment app1 \
                                    -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].image}')
                            fi

                            sed "s|image: nginx:alpine|image: $APP1_IMAGE|g" \
                                app1-deployment.yaml | kubectl apply -f -
                        fi

                        if [ "$DEPLOY_APP2" = "true" ]; then

                            if [ "$BUILD_APP2" = "true" ]; then
                                APP2_IMAGE="$DOCKERHUB_USER/$IMAGE_APP2:build-${BUILD_NUMBER}"
                            else
                                APP2_IMAGE=$(kubectl get deployment app2 \
                                    -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].image}')
                            fi

                            sed "s|image: nginx:alpine|image: $APP2_IMAGE|g" \
                                app2-deployment.yaml | kubectl apply -f -
                        fi
                    '''
                }
            }
        }

        stage('Rolling Update') {
            steps {
                sh '''
                    set -e

                    if [ "$DEPLOY_APP1" = "true" ]; then
                        kubectl rollout status deployment/app1 --timeout=120s
                    fi

                    if [ "$DEPLOY_APP2" = "true" ]; then
                        kubectl rollout status deployment/app2 --timeout=120s
                    fi
                '''
            }
        }
    }
}
