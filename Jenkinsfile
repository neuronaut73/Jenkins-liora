pipeline {
    agent any

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Movie Service') {
            steps {
                sh 'docker build -t jenkins-exam-movie:ci ./movie-service'
            }
        }

        stage('Build Cast Service') {
            steps {
                sh 'docker build -t jenkins-exam-cast:ci ./cast-service'
            }
        }

        stage('Verify Images') {
            steps {
                sh '''
                    docker image inspect jenkins-exam-movie:ci
                    docker image inspect jenkins-exam-cast:ci
                '''
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully!'
        }
        failure {
            echo 'CI pipeline failed. Check the console output.'
        }
    }
}