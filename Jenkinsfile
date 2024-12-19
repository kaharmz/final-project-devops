pipeline {

    agent any

    environment {
        GCR_HOSTNAME = "gcr.io"
        PROJECT_ID = "upheld-setting-436613-s1"
        IMAGE_NAME = "notes"
        
        BRANCH_NAME = "${env.GIT_BRANCH?.split('/')[1] ?: 'default-branch'}"
        DOCKER_IMAGE = "kaharmuzakira/${IMAGE_NAME}"
        GCR_IMAGE = "${GCR_HOSTNAME}/${PROJECT_ID}/${IMAGE_NAME}:${BRANCH_NAME}"
        GITHUB_CREDENTIALS = credentials('github-key')
        MICROK8S_KUBECONFIG = credentials('kube-key')
        GKE_CREDENTIALS = credentials('gke-key')
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
        DOCKER_AUTH = credentials('docker-auth')
    }

        stage('Clone Repository') {
            steps {
                echo "Cloning the Git repository..."
                withCredentials([sshUserPrivateKey(credentialsId: 'github-key', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                    git config --global credential.helper store
                    mkdir -p ~/.ssh
                    echo "${SSH_KEY}" > ~/.ssh/id_rsa
                    chmod 600 ~/.ssh/id_rsa
                    git clone git@github.com:kaharmz/devops-project.git .
                    """
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh """
                    set -e
                    docker build -t ${DOCKER_IMAGE} .
                    """
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                echo "Pushing Docker image to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'docker-auth', variable: 'DOCKER_AUTH')]) {
                    sh """
                    set -e
                    docker login ${DOCKER_AUTH}
                    docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Push Images to GCR') {
            steps {
                echo "Pushing Docker image to GCR..."
                withCredentials([file(credentialsId: 'gke-key', variable: 'GKE_CREDENTIALS')]) {
                    sh """
                    set -e
                    gcloud auth activate-service-account --key-file=${GKE_CREDENTIALS}
                    gcloud auth configure-docker --quiet
                    docker push ${GCR_HOSTNAME}
                    """
                }
            }
        }

        stage('Deploy to MicroK8s (Dev)') {
            when {
                branch 'dev'
            }
            steps {
                echo "Deploying to MicroK8s for branch: dev"
                writeFile file: KUBECONFIG, text: MICROK8S_KUBECONFIG
                sh """
                set -e
                export KUBECONFIG=${KUBECONFIG}
                if ! kubectl set image deployment/notes notes=${DOCKER_IMAGE} --record; then
                    kubectl apply -f devops-project/notes/notes-deployment.yaml
                fi
                """
            }
        }

        stage('Deploy to GKE (Prod)') {
            when {
                branch 'main'
            }
            steps {
                echo "Deploying to GKE for branch: main"
                withCredentials([file(credentialsId: 'gke-key', variable: 'GKE_KEY')]) {
                    sh """
                    set -e
                    kubectl set image deployment/notes notes=${GCR_HOSTNAME} --record; then
                    kubectl apply -f k8s/deployment.yaml
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up workspace..."
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs for errors."
        }
    }
}
