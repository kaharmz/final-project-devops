pipeline {
    
    agent any

    environment {
        GCR_HOSTNAME = "gcr.io"
        PROJECT_ID = "upheld-setting-436613-s1"
        IMAGE_NAME = "notes"
        BRANCH_NAME = "${env.GIT_BRANCH?.split('/')[1] ?: 'default-branch'}"
        DOCKER_IMAGE = "${GCR_HOSTNAME}/${PROJECT_ID}/${IMAGE_NAME}:${BRANCH_NAME}"

        GITHUB_CREDENTIALS = credentials('github-token')
        MICROK8S_KUBECONFIG = credentials('kube-config')
        GKE_CREDENTIALS = credentials('gke-key')
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {
        stage('Clone Repository') {
            steps {
                echo "Cloning the Git repository..."
                    sh """
                    git config --global credential.helper store
                    git clone https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/kaharmz/devops-project.git .
                    """           
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

        stage('Push Images to GCR') {
            steps {
                echo "Pushing Docker image to GCR..."
                withCredentials([file(credentialsId: 'gke-key', variable: 'GCR_KEY')]) {
                    sh """
                    set -e
                    gcloud auth activate-service-account --key-file=${GCR_KEY}
                    gcloud auth configure-docker --quiet
                    docker push ${DOCKER_IMAGE}
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
                    kubectl apply -f gke/notes-deployment.yaml
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
                    if ! kubectl set image deployment/notes notes=${DOCKER_IMAGE} --record; then
                        kubectl apply -f k8s/notes-deployment.yaml
                    fi
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
