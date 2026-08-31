pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'production'],
            description: 'Select deployment environment'
        )
    }

    environment {
        IMAGE_NAME   = 'nazeefkhan228/spring-boot-app'  
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        // Build and test are ONE step now: mvn package runs tests as part of
        // packaging, and fails the whole stage (stopping the pipeline) if any
        // test fails — no artifact gets built from code that didn't pass.
        stage('Build & Test') {
            steps {
                echo 'Building and testing Java application...'
                sh 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: '0b16352d-915d-4664-8b36-24416c33e4ef',   // TODO: create this credential in Jenkins first
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh '''
                        echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy via Ansible') {
            steps {
                echo "Deploying ${IMAGE_NAME}:${IMAGE_TAG} to ${params.ENVIRONMENT}"
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'af38c44e-b8f4-4c13-a6cb-9204de78aeb7',          // TODO: create this credential in Jenkins first
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        ansible-playbook -i inventory.ini deploy.yml \
                          --private-key="$SSH_KEY" \
                          --extra-vars "image_name=${IMAGE_NAME} image_tag=${IMAGE_TAG}"
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed — nothing was pushed or deployed past the failing stage.'
        }
    }
}
