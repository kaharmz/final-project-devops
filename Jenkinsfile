pipeline {
    agent any

    environment {
        GCR_HOSTNAME = "gcr.io"
        PROJECT_ID = "upheld-setting-436613-s1"
        IMAGE_NAME = "notes"
        BRANCH_NAME = "${env.GIT_BRANCH?.split('/')[1] ?: 'default-branch'}"
        DOCKER_IMAGE = "kaharmuzakira/${IMAGE_NAME}"
        GCR_IMAGE = "${GCR_HOSTNAME}/${PROJECT_ID}/${IMAGE_NAME}:${BRANCH_NAME}"
        SSH_KEY = credentials('github-key')
        MICROK8S_KUBECONFIG = credentials('kube-key')
        GKE_CREDENTIALS = credentials('gke-key')
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
        DOCKER_AUTH = credentials('docker-auth')
    }

    stages {
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
                withCredentials([usernamePassword(credentialsId: 'docker-auth', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    set -e
                    docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
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
                    docker push ${GCR_IMAGE}
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
                    gcloud auth activate-service-account --key-file=${GKE_KEY}
                    gcloud container clusters get-credentials my-gke-cluster --region us-central1 --project ${PROJECT_ID}
                    kubectl set image deployment/notes notes=${GCR_IMAGE} --record || kubectl apply -f k8s/deployment.yaml
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
