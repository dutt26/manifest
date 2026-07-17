pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'mysonar'
        AWS_REG      = 'ap-south-1'
        CLUSTER_NAME = 'yash-cluster-1'
        IMAGE_NAME   = 'image1'
        IMAGE_TAG    = 'latest'
        DOCKER_USER  = 'yashwanthdutt26'
        REPO_NAME    = 'zomato'
    }

    stages {

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git(
                    branch: 'master',
                    url: 'https://github.com/dutt26/Zomato-Project.git'
                )
            }
        }

        stage("Sonarqube Analysis") {
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

        stage("Quality Gates") {
            steps {
                script {
                    waitForQualityGate(
                        abortPipeline: false,
                        credentialsId: 'sonar-token'
                    )
                }
            }
        }

        stage("Install Dependencies") {
            steps {
                sh 'npm install'
            }
        }

        stage("OWASP Dependency Scan") {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --format ALL --prettyPrint',
                    odcInstallation: 'DP-Check'
                )

                dependencyCheckPublisher(
                    pattern: 'dependency-check-report.xml'
                )
            }
        }

        stage("Trivy FS Scan") {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL . > trivyfs.txt'
            }
        }

        stage("Docker Build & Cache") {
            steps {
                script {
                    sh """
                        docker buildx build \
                        --context . \
                        --file Dockerfile \
                        --tag ${IMAGE_NAME}:${IMAGE_TAG} \
                        --tag ${IMAGE_NAME}:latest \
                        --cache-from=type=local,src=/tmp/.buildx-cache \
                        --cache-to=type=local,dest=/tmp/.buildx-cache-new,mode=max \
                        --label org.opencontainers.image.title="Zomato" \
                        --label org.opencontainers.image.source="https://github.com/dutt26/Zomato-Project" \
                        --label org.opencontainers.image.version="${env.BUILD_NUMBER}" \
                        --label org.opencontainers.image.created="${env.BUILD_ID}" \
                        --load .
                    """

                    sh """
                        rm -rf /tmp/.buildx-cache

                        if [ -d "/tmp/.buildx-cache-new" ]; then
                            mv /tmp/.buildx-cache-new /tmp/.buildx-cache
                        fi
                    """
                }
            }
        }

        stage("Verify & Validate Image") {
            steps {
                sh """
                    docker images
                    docker image inspect ${IMAGE_NAME}:${IMAGE_TAG}
                    trivy image --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage("Smoke Test Container") {
            steps {
                script {
                    try {
                        sh """
                            docker run -d \
                            --name zomato-test \
                            -p 3000:3000 \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                        """

                        sleep time: 20, unit: 'SECONDS'

                        sh """
                            docker ps
                            docker logs zomato-test
                        """
                    } finally {
                        sh "docker rm -f zomato-test || true"
                    }
                }
            }
        }

        stage("Push Image to Registry") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-password') {
                        sh """
                            docker tag ${IMAGE_NAME}:${IMAGE_TAG} \
                            ${DOCKER_USER}/${REPO_NAME}:myzomatoimage
                        """

                        sh """
                            docker push ${DOCKER_USER}/${REPO_NAME}:myzomatoimage
                        """
                    }
                }
            }
        }

        stage("Deploy to Amazon EKS") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        env.AWS_ACCESS_KEY_ID     = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY
                        env.AWS_DEFAULT_REGION    = AWS_REG

                        sh "aws eks update-kubeconfig --region ${AWS_REG} --name ${CLUSTER_NAME}"
                        sh "kubectl apply -f manifest.yaml"
                        sh "kubectl rollout restart deployment/zomato-deployment"
                    }
                }
            }
        }
    }
}
