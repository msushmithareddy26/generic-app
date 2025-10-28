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
                git branch: 'main', url: 'https://github.com/msushmithareddy26/generic-app.git'
            }
        }

        stage('Git Cherry-Pick Ops') {
            steps {
                script {
                    try {
                        sh "git fetch origin"
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

        stage('Build and Push Multi-Arch Images') {
            steps {
                script {
                    try {
                        // Ensure docker buildx builder exists
                        sh "docker buildx create --use || true"

                        // Parallel build and push
                        parallel(
                            "AMD64": {
                                sh """
                                docker buildx build --platform linux/amd64 -t ${ECR_REPO}:${BUILD_NUMBER}-amd64 --load .
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-amd64
                                """
                            },
                            "ARM64": {
                                sh """
                                docker buildx build --platform linux/arm64 -t ${ECR_REPO}:${BUILD_NUMBER}-arm64 --load .
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-arm64
                                """
                            }
                        )
                    } catch (err) {
                        echo "Parallel Build/Push Error: ${err}"
                        currentBuild.result = 'ABORTED'
                        error("Aborting due to multi-arch build/push failure")
                    }
                }
            }
        }
    }
}
