
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        KUBECONFIG = '/etc/rancher/k3s/k3s.yaml'
        DOCKERHUB_USER = 'neuronaut73'
        MOVIE_IMAGE = 'neuronaut73/jenkins-exam-movie'
        CAST_IMAGE = 'neuronaut73/jenkins-exam-cast'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -e

                    docker build \
                      -t ${MOVIE_IMAGE}:${BUILD_NUMBER} \
                      ./movie-service

                    docker build \
                      -t ${CAST_IMAGE}:${BUILD_NUMBER} \
                      ./cast-service
                '''
            }
        }

        stage('Verify Images') {
            steps {
                sh '''
                    set -e

                    docker image inspect \
                      ${MOVIE_IMAGE}:${BUILD_NUMBER} > /dev/null

                    docker image inspect \
                      ${CAST_IMAGE}:${BUILD_NUMBER} > /dev/null

                    echo "Both Docker images verified."
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e
                        set +x

                        echo "$DOCKER_TOKEN" | docker login \
                          --username "$DOCKER_USER" \
                          --password-stdin

                        docker push ${MOVIE_IMAGE}:${BUILD_NUMBER}
                        docker push ${CAST_IMAGE}:${BUILD_NUMBER}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to DEV') {
            steps {
                script {
                    deployEnvironment('dev')
                }
            }
        }

        stage('Deploy to QA') {
            steps {
                script {
                    deployEnvironment('qa')
                }
            }
        }

        stage('Deploy to STAGING') {
            steps {
                script {
                    deployEnvironment('staging')
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
            echo "Docker image tag: ${env.BUILD_NUMBER}"
            echo 'Applications deployed to DEV, QA and STAGING.'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo "Build finished: ${env.BUILD_URL}"
        }
    }
}


// Reusable Helm deployment function
def deployEnvironment(String namespace) {

    echo "Deploying Movie and Cast to ${namespace}"

    sh """
        set -e

        echo "Deploying CAST to ${namespace}"

        helm upgrade --install jenkins-cast ./charts \\
          --namespace ${namespace} \\
          --set image.repository=\${CAST_IMAGE} \\
          --set-string image.tag=\${BUILD_NUMBER} \\
          --set database.host=jenkins-cast-postgres \\
          --set database.name=cast_db_dev \\
          --set database.username=cast_db_username \\
          --set database.passwordKey=cast-password \\
          --set healthPath=/api/v1/casts/docs \\
          --wait --timeout 180s

        echo "Deploying MOVIE to ${namespace}"

        helm upgrade --install jenkins-movie ./charts \\
          --namespace ${namespace} \\
          --set image.repository=\${MOVIE_IMAGE} \\
          --set-string image.tag=\${BUILD_NUMBER} \\
          --set database.host=jenkins-movie-postgres \\
          --set database.name=movie_db_dev \\
          --set database.username=movie_db_username \\
          --set database.passwordKey=movie-password \\
          --set healthPath=/api/v1/movies/docs \\
          --set-string castServiceUrl=http://jenkins-cast-fastapiapp/api/v1/casts/ \\
          --wait --timeout 180s

        echo "Checking Kubernetes deployments in ${namespace}"

        kubectl rollout status \\
          deployment/jenkins-cast-fastapiapp \\
          -n ${namespace} --timeout=120s

        kubectl rollout status \\
          deployment/jenkins-movie-fastapiapp \\
          -n ${namespace} --timeout=120s

        echo "Deployment to ${namespace} successful."
    """
}
