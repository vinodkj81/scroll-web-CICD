pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "vinodkj81/cicd:${BUILD_NUMBER}"
        DEPLOY_FILE  = 'deploy.yaml'
        DOMAIN       = 'cicd-vinod.duckdns.org'
    }

    stages {

        stage('User Confirmation') {
            steps {
                script {
                    def userInput = input(
                        id: 'userConfirm',
                        message: 'Do you want to build this project?',
                        parameters: [
                            choice(
                                name: 'CONFIRM',
                                choices: ['Yes', 'No'],
                                description: 'Select Yes to proceed or No to abort'
                            )
                        ]
                    )

                    if (userInput == 'No') {
                        echo "Build aborted by user."
                        currentBuild.result = 'ABORTED'
                        error("User chose not to proceed.")
                    }
                }
            }
        }

        stage('Select Branch') {
            steps {
                script {
                    def branchInput = input(
                        id: 'branchSelect',
                        message: 'Select the branch to build:',
                        parameters: [
                            string(
                                name: 'BRANCH',
                                defaultValue: 'master',
                                description: 'Enter the branch name to build'
                            )
                        ]
                    )

                    env.BRANCH_NAME = branchInput.trim()

                    echo "Selected Branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Clone Repository') {
            steps {
                git(
                    branch: "${env.BRANCH_NAME}",
                    url: 'https://github.com/vinodkj81/scroll-web-CICD.git'
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image: $DOCKER_IMAGE"
                    docker build -t "$DOCKER_IMAGE" .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "Pushing Docker image: $DOCKER_IMAGE"
                    docker push "$DOCKER_IMAGE"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "Deploying Kubernetes resources..."

                    kubectl apply -f "$DEPLOY_FILE"

                    echo "Updating Deployment image..."

                    kubectl set image deployment/my-deploy-app \
                        devops-app-c1="$DOCKER_IMAGE"

                    echo "Waiting for rollout..."

                    kubectl rollout status deployment/my-deploy-app \
                        --timeout=180s

                    echo "Deployment:"
                    kubectl get deployment my-deploy-app

                    echo "Pods:"
                    kubectl get pods -l app=my-app
                '''
            }
        }

        stage('Verify Ingress') {
            steps {
                sh '''
                    echo "Ingress configuration:"
                    kubectl get ingress devops-ingress

                    echo "Service:"
                    kubectl get service devops-service

                    echo "Testing Ingress locally..."

                    curl -I \
                        -H "Host: $DOMAIN" \
                        http://127.0.0.1/ \
                        || echo "Ingress local test failed. Check DNS/EC2/network settings."

                    echo "Application URL:"
                    echo "http://$DOMAIN"
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline executed successfully."
            echo "Application: http://${DOMAIN}"
        }

        failure {
            echo "Build or deployment failed. Please review the Jenkins console log."
        }

        aborted {
            echo "Pipeline aborted by user."
        }
    }
}
