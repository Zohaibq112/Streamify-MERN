pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        DOCKERHUB_USERNAME    = "${DOCKERHUB_CREDENTIALS_USR}"
        IMAGE_FRONTEND        = "${DOCKERHUB_USERNAME}/frontend"
        IMAGE_BACKEND         = "${DOCKERHUB_USERNAME}/backend"
        IMAGE_TAG             = "${GIT_COMMIT[0..7]}"
        DEPLOY_SERVER         = credentials('deploy-server-ssh')
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
                            sh 'npm ci --legacy-peer-deps'
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
                            sh 'npm run test --if-present -- --watchAll=false --passWithNoTests || true'
                        }
                    }
                }
                stage('Test Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm run test --if-present -- --passWithNoTests || true'
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
                        dir('frontend') {
                            sh 'npm audit --audit-level=high || true'
                        }
                        dir('backend') {
                            sh 'npm audit --audit-level=high || true'
                        }
                    }
                }
                stage('Trivy FS Scan') {
                    steps {
                        sh '''
                            if command -v trivy &> /dev/null; then
                                trivy fs . \
                                  --exit-code 0 \
                                  --severity HIGH,CRITICAL \
                                  --no-progress \
                                  --format table
                            else
                                echo "⚠️ Trivy not installed, skipping scan"
                            fi
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

                            echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin &&

                            docker pull ${IMAGE_FRONTEND}:latest &&
                            docker pull ${IMAGE_BACKEND}:latest &&

                            docker-compose up -d --no-build --remove-orphans &&

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