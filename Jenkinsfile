pipeline {
    agent any

    environment {
        AWS_REGION = "eu-north-1"
        AWS_ACCOUNT_ID = "527930216402"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/generic-app"
        IMAGE_TAG_AMD64 = "generic-app:${BUILD_NUMBER}-amd64"
        IMAGE_TAG_ARM64 = "generic-app:${BUILD_NUMBER}-arm64"
    }

    parameters {
        string(name: 'COMMIT_HASH', defaultValue: '', description: 'Hotfix commit hash (optional)')
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
                echo "Checking out main branch..."
                checkout([$class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/msushmithareddy26/generic-app.git',
                        credentialsId: 'github-cred' // Replace with your GitHub credential ID
                    ]]
                ])
            }
        }

        stage('Cherry-Pick Hotfix') {
            steps {
                dir('.') {
                    script {
                        if (params.COMMIT_HASH?.trim()) {
                            sh 'git config user.email "m.sushmithareddy26@gmail.com"'
                            sh 'git config user.name "msushmithareddy26"'
                            echo "Cherry-picking commit: ${params.COMMIT_HASH}"
                            sh "git fetch origin hotfix"
                            sh "git checkout main"
                            sh "git reset --hard"

                            try {
                                sh "git cherry-pick ${params.COMMIT_HASH}"
                                echo "Cherry-pick applied successfully."
                            } catch (err) {
                                def commitDetails = sh(script: "git log -1 --oneline ${params.COMMIT_HASH}", returnStdout: true).trim()
                                echo "Cherry-pick Error: ${err.getMessage()}\nCommit: ${commitDetails}"
                                currentBuild.result = 'ABORTED'
                                error("Aborting due to cherry-pick failure")
                            }
                        } else {
                            echo "No hotfix commit provided, skipping cherry-pick."
                        }
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
                stage('AMD64') {
                    steps {
                        script {
                            try {
                                sh "docker buildx build --platform linux/amd64 -t ${IMAGE_TAG_AMD64} --load ."
                            } catch (err) {
                                echo "Parallel Build Error in AMD64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("Aborting parallel builds")
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
                                echo "Parallel Build Error in ARM64: ${err}"
                                currentBuild.result = 'ABORTED'
                                error("Aborting parallel builds")
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
