
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
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
                    docker build -t ${MOVIE_IMAGE}:${BUILD_NUMBER} ./movie-service
                    docker build -t ${CAST_IMAGE}:${BUILD_NUMBER} ./cast-service
                '''
            }
        }

        stage('Verify Images') {
            steps {
                sh '''
                    docker image inspect ${MOVIE_IMAGE}:${BUILD_NUMBER} > /dev/null
                    docker image inspect ${CAST_IMAGE}:${BUILD_NUMBER} > /dev/null
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
    }

    post {
        success {
            echo 'Docker images successfully built and pushed!'
        }
        failure {
            echo 'Pipeline failed. Check console output.'
        }
    }
}
