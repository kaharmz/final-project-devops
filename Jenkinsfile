pipeline {
    agent any

    environment {
        PROJECT_ID = 'upheld-setting-436613-s1'
        GCR_IMAGE = 'gcr.io/${PROJECT_ID}/notes'
        DEV_KUBECONFIG = '/home/kaharmuzakira/.kube/config-microk8s'
        PROD_KUBECONFIG = '/home/kaharmuzakira/.kube/config-gke'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Get Version') {
            steps {
                script {
                    def version = sh(script: 'git describe --tags || git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "Version: ${version}"
                    currentBuild.displayName = version
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def branch = env.BRANCH_NAME
                    sh "docker compose build  ${GCR_IMAGE}:${env.BRANCH_NAME} ."
                }
            }
        }

        stage('Push Docker Image to GCR') {
            steps {
                script {
                    sh """
                        docker push ${GCR_IMAGE}:${env.BRANCH_NAME}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'main'
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'dev') {
                        // Deploy to MicroK8s
                        sh """
                            export KUBECONFIG=${DEV_KUBECONFIG}
                            kdev set image deployment/notes=${GCR_IMAGE}:dev
                        """
                    } else if (env.BRANCH_NAME == 'main') {
                        // Deploy to GKE
                        sh """
                            export KUBECONFIG=${PROD_KUBECONFIG}
                            kprod set image deployment/notes=${GCR_IMAGE}:main
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}