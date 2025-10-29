pipeline {
    agent any

    parameters {
        string(name: 'COMMIT_HASH', defaultValue: '', description: 'Commit hash to cherry-pick from hotfix')
    }

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "YOUR_AWS_ACCOUNT_ID"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/ecr-1"
        IMAGE_TAG_AMD64 = "generic-app:${BUILD_NUMBER}-amd64"
        IMAGE_TAG_ARM64 = "generic-app:${BUILD_NUMBER}-arm64"
    }

    stages {
        stage('Checkout') {
            steps {
                echo " Cloning main branch..."
                git branch: 'main', url: 'https://github.com/msushmithareddy26/generic-app.git'
            }
        }

        stage('Git Cherry-Pick Ops') {
            steps {
                script {
                    try {
                        sh 'git fetch origin hotfix'
                        sh "git cherry-pick ${params.COMMIT_HASH}"
                    } catch (err) {
                        echo " Cherry-pick Error: ${err.getMessage()}"
                        sh "git log -1 --oneline ${params.COMMIT_HASH} || echo 'Commit not found'"
                        currentBuild.result = 'ABORTED'
                        error("Cherry-pick failed — aborting pipeline.")
                    }
                }
            }
        }

        stage('Parallel Build Images') {
            parallel {
                stage('AMD64') {
                    steps {
                        script {
                            try {
                                sh "docker buildx build --platform linux/amd64 -t ${IMAGE_TAG_AMD64} --load ."
                            } catch (err) {
                                echo " Parallel Build Error in AMD64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("AMD64 build failed.")
                            }
                        }
                    }
                }
                stage('ARM64') {
                    steps {
                        script {
                            try {
                                sh "docker buildx build --platform linux/arm64 -t ${IMAGE_TAG_ARM64} --load ."
                            } catch (err) {
                                echo " Parallel Build Error in ARM64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("ARM64 build failed.")
                            }
                        }
                    }
                }
            }
        }

        stage('Push to ECR') {
            when { expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') } }
            steps {
                echo " Pushing images to ECR..."
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker tag ${IMAGE_TAG_AMD64} ${ECR_REPO}:${BUILD_NUMBER}-amd64
                    docker tag ${IMAGE_TAG_ARM64} ${ECR_REPO}:${BUILD_NUMBER}-arm64
                    docker push ${ECR_REPO}:${BUILD_NUMBER}-amd64
                    docker push ${ECR_REPO}:${BUILD_NUMBER}-arm64
                """
            }
        }
    }

    post {
        failure { echo " Pipeline failed." }
        aborted { echo " Pipeline aborted." }
        success { echo " Pipeline succeeded!" }
    }
}
