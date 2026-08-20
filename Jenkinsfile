pipeline {
    agent any

    tools {
        maven 'Maven-3.9'   // must match a Maven installation name configured in Jenkins > Global Tool Configuration
        jdk 'JDK-17'        // must match a JDK installation name configured in Jenkins > Global Tool Configuration
    }

    parameters {
        booleanParam(name: 'PUSH_DOCKER_IMAGE', defaultValue: false, description: 'Build and push the Docker image after a successful build')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy to the target server after a successful push (implies PUSH_DOCKER_IMAGE)')
        string(name: 'DOCKER_IMAGE_NAME', defaultValue: 'yourregistry/taskapi', description: 'Docker image name/repo (without tag) — include registry prefix, e.g. docker.io/youruser/taskapi')
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-creds'       // Jenkins credential: registry username/password
        DEPLOY_SSH_CREDENTIALS_ID = 'deploy-ssh'  // Jenkins credential: SSH private key (ed25519) for the deploy user
        DEPLOY_HOST = 'ubuntu@10.0.1.6'   // change to your actual server
        DEPLOY_DIR  = '/opt/taskapi'                 // where the compose file lives on the server
        IMAGE_TAG  = "${params.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout(false)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B -ntp clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B -ntp test'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B -ntp package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Docker Build') {
            when {
                expression { return params.PUSH_DOCKER_IMAGE }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]){
                    sh "docker build -t ${DOCKER_USER}/${IMAGE_TAG} -t ${DOCKER_USER}/${IMAGE_TAG} ."
                }
                
            }
        }

        stage('Docker Push') {
            when {
                expression { return params.PUSH_DOCKER_IMAGE }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "${DOCKER_USER}/${IMAGE_TAG}"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            when {
                expression { return params.DEPLOY }
            }
            steps {
                // Copy the deploy-only compose file to the server (build-free — it just pulls the image)
                sshagent(credentials: [env.DEPLOY_SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST} 'mkdir -p ${DEPLOY_DIR}'
                        scp -o StrictHostKeyChecking=no docker-compose.prod.yml ${DEPLOY_HOST}:${DEPLOY_DIR}/docker-compose.yml
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST} '
                            cd ${DEPLOY_DIR} && \
                            export DOCKER_USER=${DOCKER_USER} && \
                            export DOCKER_IMAGE=${params.DOCKER_IMAGE_NAME} && \
                            export IMAGE_TAG=${IMAGE_TAG} && \
                            docker compose pull && \
                            docker compose up -d && \
                            docker image prune -f
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Build #${env.BUILD_NUMBER} succeeded."
        }
        failure {
            echo "Build #${env.BUILD_NUMBER} failed. Check the stage logs above."
        }
        always {
            cleanWs()
        }
    }
}