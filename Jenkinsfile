pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "vishalsinghal07/try-app"
        DOCKER_TAG   = "${BUILD_NUMBER}"
        EKS_CLUSTER_NAME = "vishal-test-cluster"
        AWS_REGION = "ap-south-1"
    }
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'Github_credentials',
                    branch: 'main',
                    url: 'https://github.com/vishalsinghalit24-prog/Project_try.git'
            }
        }
 
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🛠️ Building Docker image..."
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }
 
        stage('Docker Compose Test') {
            steps {
                script {
                    echo "🧪 Testing with Docker Compose..."
                    sh """
                        # Clean up existing containers
                        docker-compose down -v
 
                        # Build and start services
                        docker-compose build
                        docker-compose up -d
 
                        # Wait for services to be ready
                        sleep 30
                        docker-compose ps
 
                        # Test application health
                        i=1
                        while [ \$i -le 5 ]; do
                            if curl -f http://localhost:3000 2>/dev/null; then
                                echo "✅ Application is responding"
                                break
                            fi
                            echo "⏳ Waiting for application... attempt \$i/5"
                            i=\$((i+1))
                            sleep 10
                        done
                    """
                }
            }
        }
 
        stage('Push Docker Image') {
            steps {
                script {
                    echo "📦 Pushing Docker image..."
                    docker.withRegistry('https://index.docker.io/v1/', 'DockerHub') {
                        sh """
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                    echo "✅ Docker images pushed successfully"
                }
            }
        }
 
        stage('Deploy to Kubernetes') {
            steps {
                withAWS(credentials: 'aws_cred', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🔄 Updating kubeconfig..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
 
                            echo "🚀 Deploying to Kubernetes..."
                            # Deploy MongoDB first
                            kubectl apply -f mongodb-deployement.yaml
 
                            # Update application image and deploy
                            kubectl apply -f app-deployement.yaml
                            kubectl set image deployment/lost-and-found-app \\
                                lost-and-found-app=${DOCKER_IMAGE}:${DOCKER_TAG} --record
 
 
                            echo "⏳ Waiting for deployments to complete..."
                            kubectl rollout status deployment/mongodb --timeout=300s
                            kubectl rollout status deployment/lost-and-found-app --timeout=300s
 
                            echo "📊 Deployment status:"
                            kubectl get deployments
                            kubectl get services
                            kubectl get pods
                        """
                    }
                }
            }
        }
 
        stage('Get LoadBalancer URL') {
            steps {
                withAWS(credentials: 'aws_cred', region: "${AWS_REGION}") {
                    script {
                        sh """
                            echo "🌐 Getting LoadBalancer URL..."
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
 
                            i=1
                            while [ \$i -le 10 ]; do
                                EXTERNAL_IP=\$(kubectl get service app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "")
                                EXTERNAL_HOSTNAME=\$(kubectl get service app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")
 
                                if [ ! -z "\$EXTERNAL_IP" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_IP"
                                    break
                                elif [ ! -z "\$EXTERNAL_HOSTNAME" ]; then
                                    echo "🌐 Application URL: http://\$EXTERNAL_HOSTNAME"
                                    break
                                fi
 
                                echo "⏳ Waiting for LoadBalancer... attempt \$i/10"
                                i=\$((i+1))
                                sleep 20
                            done
 
                            # Show final service status
                            kubectl get service app-service
                            echo "✅ Deployment completed successfully!"
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}
