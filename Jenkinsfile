pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'echo "Building DevOps application..."'
                sh 'ls -la'
            }
        }

        stage('Test') {
            steps {
                sh 'python3 --version'
            }
        }
    }
}
