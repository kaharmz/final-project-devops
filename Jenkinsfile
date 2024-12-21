pipeline {
    agent any

    environment {

        BUILD_TAG = "${BUILD_NUMBER}"
        APP_NAME = 'notes'
        PROJECT_ID = 'upheld-setting-436613-s1'
        GCR_IMAGE = 'gcr.io/${PROJECT_ID}/${APP_NAME}'
        DEV_KUBECONFIG = '/home/kaharmuzakira/.kube/config-microk8s'
        PROD_KUBECONFIG = '/home/kaharmuzakira/.kube/config-gke'
        
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def branch = env.BRANCH_NAME
                    sh "docker buildx build --tag ${APP_NAME}> gcr.io/${PROJECT_ID}/${APP_NAME}:${BUILD_NUMBER} ."
                }
            }
        }

        stage('Push Docker Image to GCR') {
            steps {
                script {
                    sh """                   
                        docker buildx build --push ${GCR_IMAGE}:v${BUILD_TAG}
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