pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'mysonar'

        IMAGE_NAME  = 'image1'
        IMAGE_TAG   = "${BUILD_NUMBER}"

        DOCKER_USER = 'yashwanthdutt26'
        REPO_NAME   = 'zomato'

        CONTAINER_NAME = 'zomato-container'
        APP_PORT = '3000'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/dutt26/Zomato-Project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh """
                    \$SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectName=zomato \
                      -Dsonar.projectKey=zomato \
                      -Dsonar.sources=. \
                      -Dsonar.exclusions=**/node_modules/**
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'DP-Check',
                    additionalArguments: '--scan . --format ALL'
                )

                dependencyCheckPublisher(
                    pattern: 'dependency-check-report.xml'
                )
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                . > trivyfs.txt
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${DOCKER_USER}/${REPO_NAME}:latest .
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    sh """
                    docker rm -f test-container || true

                    docker run -d \
                        --name test-container \
                        -p 3001:3000 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    sleep 20

                    docker ps

                    docker logs test-container

                    docker rm -f test-container
                    """
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {

                    withDockerRegistry(
                        credentialsId: 'docker-password',
                        url: ''
                    ) {

                        sh """
                        docker push ${DOCKER_USER}/${REPO_NAME}:latest
                        """
                    }

                }
            }
        }

        stage('Deploy Docker Container') {
            steps {
                script {

                    sh """
                    echo "Stopping existing container..."

                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true

                    echo "Pulling latest image..."

                    docker pull ${DOCKER_USER}/${REPO_NAME}:latest

                    echo "Starting container..."

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:3000 \
                        ${DOCKER_USER}/${REPO_NAME}:latest

                    docker ps
                    """

                }
            }
        }

    }

    post {

        always {
            echo 'Pipeline Finished'
        }

        success {
            echo 'Application deployed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }

        cleanup {
            sh '''
            docker image prune -f || true
            docker container prune -f || true
            '''
        }
    }
}
