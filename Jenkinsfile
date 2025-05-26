pipeline {
    agent any
    stages {
        stage('Setup') {
            steps {
                sh "pip install -r requirements.txt"
            }
        }

        stage('Test') {
            steps {
                sh "pytest"
            }
        }

        stage('Start Minikube') {
            steps {
                script {
                    sh "minikube start"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageExists = sh(script: 'docker images -q flask-app:latest', returnStdout: true).trim()
                    if (imageExists) {
                        echo "Docker image already exists, skipping build."
                    } else {
                        sh "docker build --cache-from flask-app:latest -t flask-app:latest ."
                        echo "Docker image built successfully"
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'DockerHubCreds', passwordVariable: 'dockerHubPass', usernameVariable: 'dockerHubUser')]) {
                    script {
                        def imageAlreadyPushed = sh(script: 'docker manifest inspect ${dockerHubUser}/flask-app:latest > /dev/null 2>&1', returnStatus: true)
                        if (imageAlreadyPushed == 0) {
                            echo "Image already exists in DockerHub, skipping push."
                        } else {
                            sh "echo ${dockerHubPass} | docker login -u ${dockerHubUser} --password-stdin"
                            sh "docker image tag flask-app:latest ${dockerHubUser}/flask-app:latest"
                            sh "docker push ${dockerHubUser}/flask-app:latest"
                        }
                    }
                }
            }
        }

        stage('Verify Deployment Files') {
            steps {
                script {
                    def files = sh(script: 'ls k8s', returnStdout: true).trim()
                    echo "K8s folder files:\n${files}"

                    if (!files.contains("deployment.yaml") || !files.contains("service.yaml")) {
                        error "Deployment files not found in k8s/ folder!"
                    }
                }
            }
        }

        stage('Apply Kubernetes Deployment') {
            steps {
                script {
                    def minikubeStatus = sh(script: 'minikube status', returnStdout: true).trim()
                    if (!minikubeStatus.contains("Running")) {
                        error "Minikube is not running! Cannot apply Kubernetes deployment."
                    } else {
                        sh 'kubectl apply -f k8s/deployment.yaml'
                        sh 'kubectl apply -f k8s/service.yaml'
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'kubectl get pods'
                sh 'kubectl get svc'
            }
        }

        stage('Get Service URL') {
            steps {
                script {
                    sh 'nohup kubectl port-forward service/flask-app-service 5000:5000 > /dev/null 2>&1 &'
                    echo "Application is accessible at http://localhost:5000"
                }
            }
        }
    }
}
