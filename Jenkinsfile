
pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_COLOR',
            choices: ['GREEN', 'BLUE'],
            description: 'Choose which environment should receive traffic'
        )
    }

    environment {
        IMAGE_NAME = 'assignment8-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -f Dockerfile.blue -t ${IMAGE_NAME}:blue .
                    docker build -f Dockerfile.green -t ${IMAGE_NAME}:green .
                '''
            }
        }

        stage('Transfer Images to Minikube') {
            steps {
                sh '''
                    docker save ${IMAGE_NAME}:blue -o blue.tar
                    docker save ${IMAGE_NAME}:green -o green.tar

                    docker cp blue.tar minikube:/blue.tar
                    docker cp green.tar minikube:/green.tar

                    docker exec minikube ctr -n k8s.io images import /blue.tar
                    docker exec minikube ctr -n k8s.io images import /green.tar

                    rm -f blue.tar green.tar
                '''
            }
        }

        stage('Deploy Blue and Green') {
            steps {
                sh '''
                    kubectl apply -f k8s/blue-deployment.yaml
                    kubectl apply -f k8s/green-deployment.yaml

                    kubectl rollout status deployment/blue-deployment --timeout=120s
                    kubectl rollout status deployment/green-deployment --timeout=120s
                '''
            }
        }

        stage('Verify Target Environment') {
            steps {
                sh '''
                    COLOR=$(echo "${DEPLOY_COLOR}" | tr '[:upper:]' '[:lower:]')

                    kubectl rollout status deployment/${COLOR}-deployment --timeout=120s

                    READY=$(kubectl get deployment ${COLOR}-deployment \
                        -o jsonpath='{.status.readyReplicas}')

                    DESIRED=$(kubectl get deployment ${COLOR}-deployment \
                        -o jsonpath='{.spec.replicas}')

                    if [ "$READY" != "$DESIRED" ] || [ -z "$READY" ]; then
                        echo "ERROR: ${COLOR} deployment is not fully ready!"
                        exit 1
                    fi

                    echo "${COLOR} environment is healthy."
                '''
            }
        }

        stage('Switch Traffic') {
            steps {
                sh '''
                    COLOR=$(echo "${DEPLOY_COLOR}" | tr '[:upper:]' '[:lower:]')

                    cat > service-patch.yaml <<EOF
spec:
  selector:
    app: assignment8
    version: ${COLOR}
EOF

                    kubectl patch service assignment8-service \
                        --type=merge \
                        --patch-file=service-patch.yaml

                    echo "Traffic successfully switched to ${COLOR}."
                    rm -f service-patch.yaml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Final deployment status:"
                    kubectl get deployments
                    kubectl get pods -l app=assignment8
                    kubectl get service assignment8-service
                    kubectl get endpoints assignment8-service

                    echo "Active traffic selector:"
                    kubectl get service assignment8-service \
                        -o jsonpath='{.spec.selector}'
                '''
            }
        }
    }

    post {
        success {
            echo "Blue-Green deployment completed successfully!"
        }

        failure {
            echo "Deployment failed. Check the Jenkins console output."
        }
    }
}
