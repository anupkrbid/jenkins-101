pipeline {
    agent any

    stages {
        stage('Check Docker Connection') {
            steps {
                sh 'docker version'
            }
        }

        stage('Pull Test Image') {
            steps {
                sh 'docker pull hello-world'
            }
        }
    }
}