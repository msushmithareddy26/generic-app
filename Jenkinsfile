pipeline {
    agent any

    parameters {
        string(name: 'COMMIT_HASH', defaultValue: '', description: 'Hotfix commit hash to cherry-pick')
    }

    environment {
        ECR_REPO = "527930216402.dkr.ecr.eu-north-1.amazonaws.com/ecr-1"
        AWS_REGION = "eu-north-1"
    }

    stages {

        stage('Checkout') {
            steps {
                // Clone main branch of generic-app
                git branch: 'main', url: 'https://github.com/msushmithareddy26/generic-app.git'
            }
        }

        stage('Git Cherry-Pick Ops') {
            steps {
                script {
                    try {
                        // Fetch hotfix branch and cherry-pick
                        sh "git fetch origin hotfix"
                        sh "git cherry-pick ${params.COMMIT_HASH}"
                    } catch (err) {
                        sh "git log -1 --oneline ${params.COMMIT_HASH} || echo 'Commit not found'"
                        echo "Cherry-pick Error: ${err}"
                        currentBuild.result = 'ABORTED'
                        error("Aborting pipeline due to cherry-pick failure")
                    }
                }
            }
        }

        stage('Build Multi-Arch Images') {
            steps {
                script {
                    // Ensure buildx builder exists
                    sh "docker buildx create --use || true"

                    try {
                        parallel(
                            "AMD64": {
                                stage('Build AMD64') {
                                    sh """
                                        docker buildx build --platform linux/amd64 -t ${ECR_REPO}:${BUILD_NUMBER}-amd64 --load .
                                    """
                                }
                            },
                            "ARM64": {
                                stage('Build ARM64') {
                                    sh """
                                        docker buildx build --platform linux/arm64 -t ${ECR_REPO}:${BUILD_NUMBER}-arm64 --load .
                                    """
                                }
                            }
                        )
                    } catch (err) {
                        echo "Parallel Build Error: ${err}"
                        currentBuild.result = 'ABORTED'
                        error("Aborting due to multi-arch build failure")
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'ecr-creds', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                            sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-amd64
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-arm64
                            """
                        }
                    } catch (err) {
                        echo "Push to ECR failed: ${err}"
                        currentBuild.result = 'ABORTED'
                        error("Aborting pipeline due to push failure")
                    }
                }
            }
        }
    }
}
