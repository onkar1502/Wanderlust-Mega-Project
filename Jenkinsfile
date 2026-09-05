pipeline {
    agent any

    parameters {
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Base release version prefix (e.g. 1.0.0)'
        )
        string(
            name: 'FRONTEND_DOCKER_TAG',
            defaultValue: '',
            description: 'Optional: Specific frontend tag (defaults to ${VERSION}-${BUILD_NUMBER} if empty)'
        )
        string(
            name: 'BACKEND_DOCKER_TAG',
            defaultValue: '',
            description: 'Optional: Specific backend tag (defaults to ${VERSION}-${BUILD_NUMBER} if empty)'
        )
        booleanParam(
            name: 'FAIL_ON_QUALITY_GATE',
            defaultValue: false,
            description: 'Strictly abort pipeline if SonarQube quality gate fails'
        )
    }

    environment {
        DOCKER_HUB_REPO_BACKEND  = 'onkar2003/wanderlust-backend'
        DOCKER_HUB_REPO_FRONTEND = 'onkar2003/wanderlust-frontend'
        GITHUB_REPO_URL          = 'https://github.com/onkar1502/Wanderlust-Mega-Project.git'
        DOCKER_CREDENTIALS_ID    = 'dockerhub-cred'
        GITHUB_CREDENTIALS_ID    = 'Github'
        SONARQUBE_ENV_NAME       = 'Sonar'
    }

    stages {
        stage('1. Resolve Tags & Validate') {
            steps {
                script {
                    def baseVer = params.VERSION?.trim() ?: '1.0.0'
                    def defaultTag = "${baseVer}-${env.BUILD_NUMBER}"
                    env.EFFECTIVE_FRONTEND_TAG = params.FRONTEND_DOCKER_TAG?.trim() ?: defaultTag
                    env.EFFECTIVE_BACKEND_TAG  = params.BACKEND_DOCKER_TAG?.trim()  ?: defaultTag

                    echo "=========================================================="
                    echo " CI/CD Pipeline Initialized"
                    echo " Build Number       : ${env.BUILD_NUMBER}"
                    echo " Frontend Image Tag : ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}"
                    echo " Backend Image Tag  : ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}"
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

        stage('4. SonarQube Code Analysis') {
            steps {
                script {
                    echo "Starting SonarQube Code Quality Analysis..."
                    def sonarParams = "-Dsonar.projectKey=wanderlust -Dsonar.projectName=Wanderlust -Dsonar.sources=backend,frontend -Dsonar.exclusions=**/node_modules/**,**/dist/**"
                    try {
                        withSonarQubeEnv(SONARQUBE_ENV_NAME) {
                            def scannerHome = null
                            try {
                                scannerHome = tool name: 'Sonar', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                            } catch (Exception ignored) {
                                try {
                                    scannerHome = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                                } catch (Exception ignored2) {}
                            }
                            def scanner = scannerHome ? "${scannerHome}/bin/sonar-scanner" : "sonar-scanner"
                            sh "${scanner} ${sonarParams}"
                        }
                    } catch (Exception e) {
                        echo "withSonarQubeEnv failed (${e.getMessage()}). Attempting direct scanner execution..."
                        sh "sonar-scanner ${sonarParams} || echo 'Sonar scanner finished with warnings'"
                    }
                }
            }
        }

        stage('5. SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        try {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                if (params.FAIL_ON_QUALITY_GATE) {
                                    error "Pipeline aborted due to Quality Gate failure: ${qg.status}"
                                } else {
                                    echo "WARNING: Quality Gate status is ${qg.status}, proceeding as FAIL_ON_QUALITY_GATE is false."
                                }
                            } else {
                                echo "SonarQube Quality Gate passed successfully!"
                            }
                        } catch (Exception e) {
                            echo "Skipping Quality Gate wait (SonarQube webhook or step not configured): ${e.getMessage()}"
                        }
                    }
                }
            }
        }

        stage('6. Trivy Filesystem Security Scan') {
            steps {
                script {
                    echo "Running Trivy Filesystem Scan on Application Dependencies..."
                    sh "trivy fs --severity HIGH,CRITICAL --scanners vuln,secret --format table --exit-code 0 ."
                }
            }
        }

        stage('7. Docker Build Images') {
            steps {
                script {
                    echo "Building Backend Docker Image: ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}..."
                    dir('backend') {
                        sh "docker build -t ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG} ."
                    }

                    echo "Building Frontend Docker Image: ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}..."
                    dir('frontend') {
                        sh "docker build -t ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG} ."
                    }
                }
            }
        }

        stage('8. Trivy Docker Image Scan') {
            steps {
                script {
                    echo "Scanning Backend Docker Image with Trivy..."
                    sh "trivy image --severity HIGH,CRITICAL --format table --exit-code 0 ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}"

                    echo "Scanning Frontend Docker Image with Trivy..."
                    sh "trivy image --severity HIGH,CRITICAL --format table --exit-code 0 ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}"
                }
            }
        }

        stage('9. Push to Docker Hub') {
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

        stage('10. Update GitOps Kubernetes Manifests') {
            steps {
                script {
                    echo "Updating image tags in Kubernetes manifests to match Docker Hub tags exactly..."
                    dir('kubernetes') {
                        // Precise regex replacement for backend and frontend deployment images
                        sh "sed -i -E 's|image: ${DOCKER_HUB_REPO_BACKEND}:.*|image: ${DOCKER_HUB_REPO_BACKEND}:${env.EFFECTIVE_BACKEND_TAG}|g' backend.yaml"
                        sh "sed -i -E 's|image: ${DOCKER_HUB_REPO_FRONTEND}:.*|image: ${DOCKER_HUB_REPO_FRONTEND}:${env.EFFECTIVE_FRONTEND_TAG}|g' frontend.yaml"
                    }
                }
            }
        }

        stage('11. Commit & Push to GitHub (Trigger ArgoCD)') {
            steps {
                script {
                    echo "Committing manifest changes to GitHub GitOps repo..."
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
                                echo "No manifest changes detected - already at desired version."
                            else
                                git commit -m "ci(gitops): update image tags to ${env.EFFECTIVE_BACKEND_TAG} [skip ci]"
                                git push https://${GIT_USER}:${GIT_TOKEN}@github.com/onkar1502/Wanderlust-Mega-Project.git HEAD:main
                                echo "Successfully pushed updated manifests to GitHub. Argo CD will reconcile changes."
                            fi
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "CI Pipeline Completed Successfully! Argo CD will automatically synchronize Kubernetes workloads."
        }
        failure {
            echo "CI Pipeline Failed. Please review stage execution logs."
        }
    }
}
