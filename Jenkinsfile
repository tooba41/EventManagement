
pipeline {
    agent any

    environment {
        KUBECONFIG = '/var/lib/jenkins/.kube/eventhub-config'
        NAMESPACE = 'eventhub'
    }

    options {
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Checking out EventHub source code...'
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    set -e
                    docker --version
                    kubectl get nodes
                    sudo -n -u ec2-user /usr/local/bin/minikube status
                '''
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                    docker build \
                    -t eventhub-backend:${BUILD_NUMBER} \
                    ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    docker build \
                    -t eventhub-frontend:${BUILD_NUMBER} \
                    ./frontend
                '''
            }
        }

        stage('Load Images into Minikube') {
            steps {
                sh '''
                    sudo -n -u ec2-user /usr/local/bin/minikube image load eventhub-backend:${BUILD_NUMBER}

                    sudo -n -u ec2-user /usr/local/bin/minikube image load eventhub-frontend:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -e

                    kubectl apply -f k8s/postgres.yaml -n ${NAMESPACE}
                    kubectl rollout status deployment/eventhub-postgres \
                    -n ${NAMESPACE} --timeout=180s

                    kubectl apply -f k8s/backend.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/frontend.yaml -n ${NAMESPACE}

                    BACKEND_CONTAINER=$(kubectl get deployment eventhub-backend \
                    -n ${NAMESPACE} \
                    -o jsonpath='{.spec.template.spec.containers[0].name}')

                    FRONTEND_CONTAINER=$(kubectl get deployment eventhub-frontend \
                    -n ${NAMESPACE} \
                    -o jsonpath='{.spec.template.spec.containers[0].name}')

                    kubectl set image deployment/eventhub-backend \
                    ${BACKEND_CONTAINER}=eventhub-backend:${BUILD_NUMBER} \
                    -n ${NAMESPACE}

                    kubectl set image deployment/eventhub-frontend \
                    ${FRONTEND_CONTAINER}=eventhub-frontend:${BUILD_NUMBER} \
                    -n ${NAMESPACE}
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl rollout status deployment/eventhub-backend \
                    -n ${NAMESPACE} --timeout=180s

                    kubectl rollout status deployment/eventhub-frontend \
                    -n ${NAMESPACE} --timeout=180s

                    kubectl get pods,services -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: EventHub deployed successfully!'
        }

        failure {
            echo 'FAILED: Please check the Jenkins console output.'
        }

        always {
            echo 'EventHub CI/CD Pipeline finished.'
        }
    }
}
