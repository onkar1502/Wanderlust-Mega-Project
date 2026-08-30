pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Global release version tag (e.g. 1.0.1, 1.0.2)'
        )
        string(
            name: 'FRONTEND_DOCKER_TAG',
            defaultValue: '',
            description: 'Optional: Specific frontend tag (defaults to VERSION if blank)'
        )
        string(
            name: 'BACKEND_DOCKER_TAG',
            defaultValue: '',
            description: 'Optional: Specific backend tag (defaults to VERSION if blank)'
        )
    }

    environment {
        DOCKER_HUB_REPO_BACKEND  = 'onkar1502/wanderlust-backend'
        DOCKER_HUB_REPO_FRONTEND = 'onkar1502/wanderlust-frontend'
        GITHUB_REPO_URL          = 'https://github.com/onkar1502/Wanderlust-Mega-Project.git'
        DOCKER_CREDENTIALS_ID    = 'dockerhub-cred'
        GITHUB_CREDENTIALS_ID    = 'github-cred'
    }

    stages {
        stage('1. Resolve Tags & Validate') {
            steps {
                script {
                    def baseVer = params.VERSION?.trim() ?: '1.0.0'
                    env.EFFECTIVE_FRONTEND_TAG = params.FRONTEND_DOCKER_TAG?.trim() ?: baseVer
                    env.EFFECTIVE_BACKEND_TAG  = params.BACKEND_DOCKER_TAG?.trim()  ?: baseVer

                    echo "=========================================================="
                    echo "CI Pipeline Triggered"
                    echo "Frontend Image Tag : ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}"
                    echo "Backend Image Tag  : ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}"
                    echo "=========================================================="
                }
            }
        }

        stage('2. Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('3. Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('4. Docker Build Images') {
            steps {
                script {
                    echo "Building Backend Docker Image (${env.EFFECTIVE_BACKEND_TAG})..."
                    dir('backend') {
                        sh "docker build -t ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG} ."
                    }

                    echo "Building Frontend Docker Image (${env.EFFECTIVE_FRONTEND_TAG})..."
                    dir('frontend') {
                        sh "docker build -t ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG} ."
                    }
                }
            }
        }

        stage('5. Push to Docker Hub') {
            steps {
                script {
                    echo "Logging into Docker Hub and pushing images..."
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker push ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}"
                        sh "docker push ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}"
                        sh "docker logout"
                    }
                }
            }
        }

        stage('6. Update GitOps Manifests') {
            steps {
                script {
                    echo "Updating image tags in Kubernetes manifests..."
                    dir('kubernetes') {
                        // Safely replace image line without touching replicas:
                        sh "sed -i -E 's|image: ${DOCKER_HUB_REPO_BACKEND}:.*|image: ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}|g' backend.yaml"
                        sh "sed -i -E 's|image: ${DOCKER_HUB_REPO_FRONTEND}:.*|image: ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}|g' frontend.yaml"
                    }
                }
            }
        }

        stage('7. Commit & Push to GitHub') {
            steps {
                script {
                    echo "Committing GitOps manifest updates..."
                    withCredentials([usernamePassword(
                        credentialsId: "${GITHUB_CREDENTIALS_ID}",
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                            git config user.email "jenkins-ci@wanderlust.local"
                            git config user.name "Jenkins CI"
                            
                            git add kubernetes/backend.yaml kubernetes/frontend.yaml
                            
                            if git diff --staged --quiet; then
                                echo "No manifest changes detected."
                            else
                                git commit -m "ci(gitops): update image tags (backend:${env.EFFECTIVE_BACKEND_TAG}, frontend:${env.EFFECTIVE_FRONTEND_TAG}) [skip ci]"
                                git push https://${GIT_USER}:${GIT_TOKEN}@github.com/onkar1502/Wanderlust-Mega-Project.git HEAD:main
                                echo "Successfully pushed updated manifests to GitHub."
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "CI Pipeline Completed Successfully! Argo CD will automatically synchronize Kubernetes."
        }
        failure {
            echo "CI Pipeline Failed. Please check stage logs."
        }
    }
}
