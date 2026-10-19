pipeline {
    agent {
        label 'ubuntu-agent-2'
    }

    environment {
        IMAGE = "viflash/aws-devops-demo:${BUILD_NUMBER}"
        MINIKUBE_HOST = 'vijay@192.168.56.101'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', url: 'https://github.com/viflash/aws-springboot-devops.git'
            }
        }

        stage('Build and Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def image = docker.build(IMAGE)
                        image.push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent(credentials: ['ubuntu-agentVM1-credentials']) {
                    sh '''
                        set -eu
                        remote_dir="/tmp/aws-devops-demo-${BUILD_NUMBER}"
                        ssh -o StrictHostKeyChecking=no "$MINIKUBE_HOST" "mkdir -p '$remote_dir'"
                        scp -o StrictHostKeyChecking=no -r k8s/. "$MINIKUBE_HOST:$remote_dir/"
                        ssh -o StrictHostKeyChecking=no "$MINIKUBE_HOST" \
                            "set -eu
                            sed 's|IMAGE_PLACEHOLDER|$IMAGE|g' '$remote_dir/deployment.yaml' > '$remote_dir/deployment-rendered.yaml'
                            kubectl apply -f '$remote_dir/deployment-rendered.yaml'
                            kubectl apply -f '$remote_dir/service.yaml'
                            kubectl rollout status deployment/aws-devops-demo --timeout=180s"
                    '''
                }
            }
        }
    }
}