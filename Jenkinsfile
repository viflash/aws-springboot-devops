
pipeline {
    agent {
        label 'ubuntu-agent-2'
    }

    environment {
        IMAGE = "viflash/aws-devops-demo:${BUILD_NUMBER}"
        MINIKUBE_HOST = '192.168.56.101'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop', url: 'https://github.com/viflash/aws-springboot-devops.git'
            }
        }

        stage('Build and Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin

                        docker build -t "$IMAGE" .
                        docker push "$IMAGE"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ubuntu-agentVM1-credentials',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        set -eu
                        grep -q 'IMAGE_PLACEHOLDER' k8s/deployment.yaml
                        sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}|g" k8s/deployment.yaml
                        remote_dir="/tmp/aws-devops-demo-${BUILD_NUMBER}"
                        
                        SSH_OPTS="-i $SSH_KEY -o StrictHostKeyChecking=no"

                        ssh $SSH_OPTS "$SSH_USER@$MINIKUBE_HOST" "mkdir -p '$remote_dir'"
                        scp $SSH_OPTS -r k8s/. "$SSH_USER@$MINIKUBE_HOST:$remote_dir/"
                        ssh $SSH_OPTS "$SSH_USER@$MINIKUBE_HOST" \
                            "set -eu
                            kubectl apply -f '$remote_dir/'
                            kubectl rollout status deployment/aws-devops-demo --timeout=180s"
                    '''
                }
            }
        }
    }
}
