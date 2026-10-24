
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        KUBECONFIG = '/etc/rancher/k3s/k3s.yaml'
        IMAGE_TAG = "${env.JOB_NAME}-${env.BUILD_NUMBER}"
        DOCKERHUB_USER = 'neuronaut73'
        MOVIE_IMAGE = 'neuronaut73/jenkins-exam-movie'
        CAST_IMAGE = 'neuronaut73/jenkins-exam-cast'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "Jenkins job: ${JOB_NAME}"
                    echo "Git revision:"
                    git rev-parse HEAD
                    echo "Git remote branches:"
                    git branch -r
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -e

                    docker build \
                      -t ${MOVIE_IMAGE}:${IMAGE_TAG} \
                      ./movie-service

                    docker build \
                      -t ${CAST_IMAGE}:${IMAGE_TAG} \
                      ./cast-service
                '''
            }
        }

        stage('Verify Images') {
            steps {
                sh '''
                    set -e

                    docker image inspect \
                      ${MOVIE_IMAGE}:${IMAGE_TAG} > /dev/null

                    docker image inspect \
                      ${CAST_IMAGE}:${IMAGE_TAG} > /dev/null

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

                        docker push ${MOVIE_IMAGE}:${IMAGE_TAG}
                        docker push ${CAST_IMAGE}:${IMAGE_TAG}

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

        stage('Approve Production') {
            when {
                expression {
                    return isProductionBranch()
                }
            }

            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: 'Deploy Movie and Cast to PRODUCTION?',
                        ok: 'Approve Deployment',
                        submitter: 'jenkins'
                    )
                }
            }
        }

        stage('Deploy to PROD') {
            when {
                expression {
                    return isProductionBranch()
                }
            }

            steps {
                script {
                    deployEnvironment('prod')
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
            echo "Docker image tag: ${env.IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        aborted {
            echo 'Pipeline aborted or production approval timed out.'
        }

        always {
            echo "Build finished: ${env.BUILD_URL}"
        }
    }
}


// Verify that this is the production job
// AND the checked-out commit belongs to origin/master.
def isProductionBranch() {

    if (env.JOB_NAME != 'Jenkins-Liora-Production') {
        echo 'Production skipped: not the production Jenkins job.'
        return false
    }

    def result = sh(
        script: '''
            set -e

            git fetch origin master

            test "$(git rev-parse HEAD)" = \
                 "$(git rev-parse refs/remotes/origin/master)"
        ''',
        returnStatus: true
    )

    if (result != 0) {
        echo 'Production skipped: checked-out commit is not origin/master.'
        return false
    }

    echo 'Production branch verification successful.'
    return true
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
          --set-string image.tag=\${IMAGE_TAG} \\
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
          --set-string image.tag=\${IMAGE_TAG} \\
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
