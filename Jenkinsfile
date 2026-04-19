pipeline {
    agent any

    environment {
        DOCKERHUB_REGISTRY = 'virendranawkar'
        IMAGE_NAME         = 'todo-app'
        IMAGE_TAG          = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        HELM_CHART         = './todo-helm'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker build -t $DOCKERHUB_REGISTRY/$IMAGE_NAME:$IMAGE_TAG .'
                    sh 'docker push $DOCKERHUB_REGISTRY/$IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                script {
                    def branch = env.BRANCH_NAME

                    if (branch == 'develop') {
                        NAMESPACE    = 'dev'
                        VALUES_FILE  = 'todo-helm/values-dev.yaml'
                        RELEASE_NAME = 'todo-dev'

                    } else if (branch == 'staging') {
                        NAMESPACE    = 'staging'
                        VALUES_FILE  = 'todo-helm/values-staging.yaml'
                        RELEASE_NAME = 'todo-staging'

                    } else if (branch == 'main') {
                        NAMESPACE    = 'production'
                        VALUES_FILE  = 'todo-helm/values-prod.yaml'
                        RELEASE_NAME = 'todo-prod'

                    } else {
                        echo "Branch '${branch}' is not a deploy branch. Skipping deploy."
                        return
                    }

                    echo "Deploying branch '${branch}' to namespace '${NAMESPACE}' with release '${RELEASE_NAME}'"

                    sh "helm upgrade --install ${RELEASE_NAME} ${HELM_CHART} --namespace ${NAMESPACE} --create-namespace -f ${VALUES_FILE} --set image.tag=${IMAGE_TAG} --wait --timeout 5m"
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker rmi $DOCKERHUB_REGISTRY/$IMAGE_NAME:$IMAGE_TAG || true'
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Branch: ${env.BRANCH_NAME} | Image: ${env.DOCKERHUB_REGISTRY}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            echo "Pipeline FAILED — Branch: ${env.BRANCH_NAME}"
        }
    }
}
