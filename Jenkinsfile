pipeline {
    agent any

    environment {
        AWS_REGION = "eu-north-1"
        AWS_ACCOUNT_ID = "527930216402"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/ecr-1"
        IMAGE_TAG_AMD64 = "generic-app:${BUILD_NUMBER}-amd64"
        IMAGE_TAG_ARM64 = "generic-app:${BUILD_NUMBER}-arm64"
    }

    parameters {
        string(name: 'HOTFIX_COMMIT', defaultValue: '', description: 'Commit hash to cherry-pick from hotfix branch')
    }

    stages {
        stage('Clean Workspace') {
            steps {
                deleteDir()
                echo "Workspace cleaned."
            }
        }

        stage('Checkout') {
            steps {
                echo "Cloning main branch..."
                git branch: 'main', url: 'https://github.com/msushmithareddy26/generic-app.git'
            }
        }

        stage('Git Cherry-Pick Hotfix') {
            steps {
                script {
                    try {
                        echo "Cherry-picking commit ${params.HOTFIX_COMMIT} from hotfix branch..."
                        sh "git fetch origin hotfix"
                        sh "git checkout main"
                        sh "git reset --hard"
                        sh "git cherry-pick ${params.HOTFIX_COMMIT}"
                    } catch (err) {
                        echo "Cherry-pick Error: ${err.getMessage()}"
                        sh "git log -1 --oneline ${params.HOTFIX_COMMIT} || echo 'Commit not found'"
                        currentBuild.result = 'ABORTED'
                        error("Aborting pipeline due to cherry-pick failure")
                    }
                }
            }
        }

        stage('Build Multi-Arch Images') {
            steps {
                script {
                    sh "docker buildx create --use || true"
                }
            }

            parallel {
                stage('Build AMD64') {
                    steps {
                        script {
                            try {
                                sh "docker buildx build --platform linux/amd64 -t ${IMAGE_TAG_AMD64} --load ."
                            } catch (err) {
                                echo "Parallel Build Error in AMD64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("Aborting pipeline due to AMD64 build failure")
                            }
                        }
                    }
                }

                stage('Build ARM64') {
                    steps {
                        script {
                            try {
                                sh "docker buildx build --platform linux/arm64 -t ${IMAGE_TAG_ARM64} --load ."
                            } catch (err) {
                                echo "Parallel Build Error in ARM64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("Aborting pipeline due to ARM64 build failure")
                            }
                        }
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    try {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                        sh "docker tag ${IMAGE_TAG_AMD64} ${ECR_REPO}:${BUILD_NUMBER}-amd64"
                        sh "docker tag ${IMAGE_TAG_ARM64} ${ECR_REPO}:${BUILD_NUMBER}-arm64"
                        sh "docker push ${ECR_REPO}:${BUILD_NUMBER}-amd64"
                        sh "docker push ${ECR_REPO}:${BUILD_NUMBER}-arm64"
                        echo "Images pushed to ECR successfully."
                    } catch (err) {
                        echo "ECR Push Error: ${err}"
                        currentBuild.result = 'ABORTED'
                        error("Aborting pipeline due to ECR push failure")
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.result}"
        }
    }
}
