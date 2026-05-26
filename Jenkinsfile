pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')   // Jenkins credential ID
        DOCKERHUB_USERNAME    = "${DOCKERHUB_CREDENTIALS_USR}"
        IMAGE_FRONTEND        = "${DOCKERHUB_USERNAME}/frontend"
        IMAGE_BACKEND         = "${DOCKERHUB_USERNAME}/backend"
        IMAGE_TAG             = "${GIT_COMMIT[0..7]}"            // short commit SHA
        DEPLOY_SERVER = "ubuntu"
        DEPLOY_HOST = "192.168.1.10" // SSH credential ID
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        // ─────────────────────────────────────────
        stage('Code Push') {
        // ─────────────────────────────────────────
            steps {
                echo "✅ Triggered by commit: ${GIT_COMMIT}"
                echo "📦 Branch: ${GIT_BRANCH}"
                sh 'git log -1 --pretty="%h %an %s"'
            }
        }

        // ─────────────────────────────────────────
        stage('Build') {
        // ─────────────────────────────────────────
            parallel {
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                            sh 'npm run build'
                        }
                    }
                }
                stage('Build Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm ci'
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Test') {
        // ─────────────────────────────────────────
            parallel {
                stage('Test Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm test -- --watchAll=false --passWithNoTests'
                        }
                    }
                }
                stage('Test Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm test -- --passWithNoTests'
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Security Scan') {
        // ─────────────────────────────────────────
            parallel {
                stage('Dependency Audit') {
                    steps {
                        dir('frontend') { sh 'npm audit --audit-level=high' }
                        dir('backend')  { sh 'npm audit --audit-level=high' }
                    }
                }
                stage('Trivy FS Scan') {
                    steps {
                        sh '''
                            trivy fs . \
                              --exit-code 1 \
                              --severity HIGH,CRITICAL \
                              --no-progress \
                              --format table
                        '''
                    }
                }
            }
        }

        // ─────────────────────────────────────────
        stage('Docker Build') {
        // ─────────────────────────────────────────
            parallel {
                stage('Build Frontend Image') {
                    steps {
                        dir('frontend') {
                            sh """
                                docker build \\
                                  --build-arg VITE_API_URL=\${VITE_API_URL} \\
                                  --build-arg VITE_APP_ENV=production \\
                                  -t ${IMAGE_FRONTEND}:${IMAGE_TAG} \\
                                  -t ${IMAGE_FRONTEND}:latest \\
                                  .
                            """
                        }
                    }
                }
                stage('Build Backend Image') {
                    steps {
                        dir('backend') {
                            sh """
                                docker build \\
                                  -t ${IMAGE_BACKEND}:${IMAGE_TAG} \\
                                  -t ${IMAGE_BACKEND}:latest \\
                                  .
                            """
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────
     // ─────────────────────────────────────────
        stage('Push Image') {
        // ─────────────────────────────────────────
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh "docker push ${IMAGE_FRONTEND}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_FRONTEND}:latest"
                sh "docker push ${IMAGE_BACKEND}:${IMAGE_TAG}"
                sh "docker push ${IMAGE_BACKEND}:latest"
            }
        }

        // ─────────────────────────────────────────
        stage('Deploy') {
        // ─────────────────────────────────────────
            steps {
                sshagent(credentials: ['deploy-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER}@your-server-ip '
                            cd /opt/your-app &&

                            # Log in to Docker Hub on the server
                            echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin &&

                            # Pull latest images
                            docker pull ${IMAGE_FRONTEND}:latest &&
                            docker pull ${IMAGE_BACKEND}:latest &&

                            # Restart containers with new images (zero-downtime)
                            docker-compose up -d --no-build --remove-orphans &&

                            # Clean up old dangling images
                            docker image prune -f
                        '
                    """
                }
            }
        }
    }

    // ─────────────────────────────────────────
    post {
    // ─────────────────────────────────────────
        success {
            echo "🚀 Deployment successful! Tag: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check logs above."
        }
        always {
            // Clean up local docker images to save disk space
            sh """
                docker rmi ${IMAGE_FRONTEND}:${IMAGE_TAG} || true
                docker rmi ${IMAGE_BACKEND}:${IMAGE_TAG}  || true
                docker rmi ${IMAGE_FRONTEND}:latest       || true
                docker rmi ${IMAGE_BACKEND}:latest        || true
            """
            cleanWs()
        }
    }
}