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
        IMAGE_NAME = 'nazeefkhan228/spring-boot-app'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        stage('Build & Test') {
            steps {
                echo 'Building and testing Java application...'
                sh 'mvn clean package'
            }
        }
        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: '0b16352d-915d-4664-8b36-24416c33e4ef',
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
                    credentialsId: 'af38c44e-b8f4-4c13-a6cb-9204de78aeb7',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        # Copy the injected key into the workspace so the container can see it,
                        # then lock its permissions down — SSH refuses keys that are too open.
                        cp "$SSH_KEY" "$WORKSPACE/deploy_key"
                        chmod 600 "$WORKSPACE/deploy_key"

                        docker run --rm \
                          --network host \
                          --user root \
                          -e ANSIBLE_HOST_KEY_CHECKING=False \
                          -v "$WORKSPACE:/ansible" \
                          -w /ansible \
                          quay.io/ansible/ansible-runner \
                          ansible-playbook -i inventory.ini deploy.yml \
                            --private-key=/ansible/deploy_key \
                            --extra-vars "image_name=${IMAGE_NAME} image_tag=${IMAGE_TAG}"

                        # Don't leave the key sitting in the workspace after the run.
                        rm -f "$WORKSPACE/deploy_key"
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
