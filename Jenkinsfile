
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

                    echo "Checking Docker..."
                    docker --version

                    echo "Checking Kubernetes access..."
                    kubectl get pods -n ${NAMESPACE}

                    echo "Checking deployment permissions..."
                    kubectl auth can-i create deployments -n ${NAMESPACE}
                '''
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                    set -e

                    echo "Building EventHub backend image..."

                    docker build \
                    -t eventhub-backend:${BUILD_NUMBER} \
                    ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    set -e

                    echo "Building EventHub frontend image..."

                    docker build \
                    -t eventhub-frontend:${BUILD_NUMBER} \
                    ./frontend
                '''
            }
        }

        stage('Load Images into Minikube') {
            steps {
                sh '''
                    set -e

                    echo "Loading backend image..."

                    sudo -n -u ec2-user /usr/local/sbin/jenkins-minikube-image-load eventhub-backend:${BUILD_NUMBER}

                    echo "Loading frontend image..."

                    sudo -n -u ec2-user /usr/local/sbin/jenkins-minikube-image-load eventhub-frontend:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                sh '''
                    set -e

                    echo "Deploying PostgreSQL..."

                    kubectl apply -f k8s/postgres.yaml -n ${NAMESPACE}

                    kubectl rollout status deployment/eventhub-postgres \
                    -n ${NAMESPACE} --timeout=180s
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                sh '''
                    set -e

                    echo "Deploying backend..."

                    kubectl apply -f k8s/backend.yaml -n ${NAMESPACE}

                    BACKEND_CONTAINER=$(kubectl get deployment eventhub-backend \
                    -n ${NAMESPACE} \
                    -o jsonpath='{.spec.template.spec.containers[0].name}')

                    kubectl set image deployment/eventhub-backend \
                    ${BACKEND_CONTAINER}=eventhub-backend:${BUILD_NUMBER} \
                    -n ${NAMESPACE}

                    kubectl rollout status deployment/eventhub-backend \
                    -n ${NAMESPACE} --timeout=180s
                '''
            }
        }

        stage('Deploy Frontend') {
            steps {
                sh '''
                    set -e

                    echo "Deploying frontend..."

                    kubectl apply -f k8s/frontend.yaml -n ${NAMESPACE}

                    FRONTEND_CONTAINER=$(kubectl get deployment eventhub-frontend \
                    -n ${NAMESPACE} \
                    -o jsonpath='{.spec.template.spec.containers[0].name}')

                    kubectl set image deployment/eventhub-frontend \
                    ${FRONTEND_CONTAINER}=eventhub-frontend:${BUILD_NUMBER} \
                    -n ${NAMESPACE}

                    kubectl rollout status deployment/eventhub-frontend \
                    -n ${NAMESPACE} --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "Checking EventHub pods..."

                    kubectl get pods -n ${NAMESPACE}

                    echo "Checking EventHub services..."

                    kubectl get services -n ${NAMESPACE}

                    echo "Checking EventHub deployments..."

                    kubectl get deployments -n ${NAMESPACE}
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
