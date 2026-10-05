pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-2'
        ECR_REPOSITORY = 'devops-end-to-end'
        SONARQUBE_SERVER = 'SonarQube'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Configure AWS') {
            steps {
                script {
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query Account --output text',
                        returnStdout: true
                    ).trim()

                    env.ECR_REGISTRY =
                        "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                }

                sh '''
                    set -eu

                    echo "AWS Region: $AWS_REGION"
                    echo "ECR Repository: $ECR_REPOSITORY"

                    aws sts get-caller-identity

                    aws ecr describe-repositories \
                      --repository-names "$ECR_REPOSITORY" \
                      --region "$AWS_REGION"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -eu
                    python3 -m venv .venv
                    . .venv/bin/activate
                    python -m pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    set -eu
                    . .venv/bin/activate
                    python -m unittest discover -s tests -v
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        set -eu
                        sonar-scanner \
                          -Dsonar.projectKey=devops-end-to-end \
                          -Dsonar.sources=. \
                          -Dsonar.python.version=3.12
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      -t "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER" \
                      -t "$ECR_REGISTRY/$ECR_REPOSITORY:latest" \
                      .

                    docker images
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                sh '''
                    set -eu

                    aws ecr get-login-password \
                      --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    set -eu

                    docker push "$ECR_REGISTRY/$ECR_REPOSITORY:$BUILD_NUMBER"
                    docker push "$ECR_REGISTRY/$ECR_REPOSITORY:latest"
                '''
            }
        }

        stage('Verify Image in ECR') {
            steps {
                sh '''
                    set -eu

                    aws ecr describe-images \
                      --repository-name "$ECR_REPOSITORY" \
                      --region "$AWS_REGION" \
                      --query 'imageDetails[*].imageTags' \
                      --output json
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Tests passed and the Docker image was pushed to ECR.'
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output for the first error.'
        }

        always {
            echo 'Jenkins pipeline finished.'
        }
    }
}
