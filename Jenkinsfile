pipeline {
    agent any

    environment {
        DOCKER_IMAGE  = "simbudevops/devopsexamapp:latest"
        EKS_CLUSTER   = "simbu-cluster"    
        K8S_NAMESPACE = "devopsexamapp"
        AWS_REGION    = "ap-south-1"        
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/simbudevops/3-tier-application.git',
                    branch: 'master'
            }
        }

        stage('Verify Docker Compose') {
            steps {
                sh '''
                docker compose version || { echo "Docker Compose not available"; exit 1; }
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                    echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                    docker push ${DOCKER_IMAGE}
                    docker logout
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                docker compose down --remove-orphans || true
                docker compose up -d --build

                echo "Waiting for MySQL to be ready..."
                timeout 120s bash -c '
                while ! docker compose exec -T mysql mysqladmin ping -uroot -prootpass --silent;
                do
                    sleep 5
                    docker compose logs mysql --tail=5 || true
                done'

                sleep 10
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                echo "=== Container Status ==="
                docker compose ps -a
                echo "=== Testing Flask Endpoint ==="
                curl -I http://localhost:5000 || true
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-creds',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'],
                    usernamePassword(credentialsId: 'dockerhub-creds',
                                     usernameVariable: 'DH_USER',
                                     passwordVariable: 'DH_PASS')
                ]) {
                    sh '''
                    aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}

                    kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                    kubectl create secret docker-registry dockerhub-creds \
                        --docker-server=https://index.docker.io/v1/ \
                        --docker-username="$DH_USER" \
                        --docker-password="$DH_PASS" \
                        --namespace=${K8S_NAMESPACE} \
                        --dry-run=client -o yaml | kubectl apply -f -

                    kubectl apply -f deployment.yml -n ${K8S_NAMESPACE}
                    kubectl apply -f service.yml -n ${K8S_NAMESPACE}

                    kubectl rollout status deployment/devopsexamapp -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '🚀 Deployment successful!'
            sh 'docker compose ps'
            sh 'docker images | grep devopsexamapp || true'
        }
        failure {
            echo '❗ Pipeline failed. Check logs above.'
            sh '''
            echo "=== Error Investigation ==="
            docker compose logs --tail=50 || true
            '''
        }
        always {
            sh '''
            echo "=== Final Logs ==="
            docker compose logs --tail=20 || true
            '''
        }
    }
}
