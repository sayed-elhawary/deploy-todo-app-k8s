pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub')
        KUBECONFIG = credentials('kubeconfig-todo-app')
        DOCKERHUB_USERNAME = 'sayedkhaledelhawary'
        GIT_REPO = 'https://github.com/sayed-elhawary/deploy-todo-app-k8s.git'
        SONARQUBE_ENV = 'MySonarQubeServer'
    }

    triggers {
        pollSCM('* * * * *')
    }

    stages {
        stage('Clone Git Repo') {
            steps {
                git url: "${GIT_REPO}", branch: 'main'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            docker run --rm \
                            -e SONAR_HOST_URL=$SONAR_HOST_URL \
                            -e SONAR_LOGIN=$SONAR_TOKEN \
                            -v ${WORKSPACE}:/usr/src \
                            sonarsource/sonar-scanner-cli:5.0.1 \
                            sonar-scanner \
                                -Dsonar.projectKey=todo-app \
                                -Dsonar.sources=./backend,./frontend \
                                -Dsonar.exclusions=**/node_modules/**,**/build/**,**/dist/** \
                                -Dsonar.sourceEncoding=UTF-8
                        """
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def version = "v1.${env.BUILD_NUMBER}"
                    sh "docker build -t $DOCKERHUB_USERNAME/frontend:${version} -t $DOCKERHUB_USERNAME/frontend:latest ./frontend"
                    sh "docker build -t $DOCKERHUB_USERNAME/backend:${version} -t $DOCKERHUB_USERNAME/backend:latest ./backend"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    def version = "v1.${env.BUILD_NUMBER}"
                    sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_USERNAME --password-stdin"
                    sh "docker push $DOCKERHUB_USERNAME/frontend:${version}"
                    sh "docker push $DOCKERHUB_USERNAME/frontend:latest"
                    sh "docker push $DOCKERHUB_USERNAME/backend:${version}"
                    sh "docker push $DOCKERHUB_USERNAME/backend:latest"
                    sh "docker logout"
                }
            }
        }

        stage('Deploy Prometheus & Grafana') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-todo-app', variable: 'KUBECONFIG')]) {
                    sh '''
                        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                        helm repo add grafana https://grafana.github.io/helm-charts || true
                        helm repo update

                        kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

                        helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
                            --namespace monitoring \
                            --set prometheus.service.type=NodePort \
                            --set prometheus.service.nodePort=32002 \
                            --set grafana.service.type=NodePort \
                            --set grafana.service.nodePort=32001 \
                            --set grafana.adminPassword=admin
                    '''
                }
            }
        }

        stage('Apply ServiceMonitor') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-todo-app', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl get crd servicemonitors.monitoring.coreos.com || echo "❗ ServiceMonitor CRD not installed"
                        kubectl apply -f kubernetes/backend-servicemonitor.yaml -n monitoring
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-todo-app', variable: 'KUBECONFIG')]) {
                    script {
                        sh '''
                            kubectl set image deployment/frontend frontend=$DOCKERHUB_USERNAME/frontend:latest -n todo-app || true
                            kubectl set image deployment/backend backend=$DOCKERHUB_USERNAME/backend:latest -n todo-app || true
                            kubectl rollout restart deployment frontend -n todo-app
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline finished."
        }
    }
}
