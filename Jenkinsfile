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

        stage('Build Multi-Arch Images') {
            steps {
                script {
                    try {
                        sh "docker buildx create --use || true"

                        parallel(
                            "AMD64 Build": {
                                script {
                                    try {
                                        sh "docker buildx build --platform linux/amd64 -t ${ECR_REPO}:${BUILD_NUMBER}-amd64 --load ."
                                    } catch (err) {
                                        echo "AMD64 Build failed: ${err}"
                                        error("Aborting due to AMD64 build failure")
                                    }
                                }
                            },
                            "ARM64 Build": {
                                script {
                                    try {
                                        sh "docker buildx build --platform linux/arm64 -t ${ECR_REPO}:${BUILD_NUMBER}-arm64 --load ."
                                    } catch (err) {
                                        echo "ARM64 Build failed: ${err}"
                                        error("Aborting due to ARM64 build failure")
                                    }
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

        stage('Push Multi-Arch Images to ECR') {
            steps {
                script {
                    parallel(
                        "Push AMD64": {
                            withCredentials([usernamePassword(credentialsId: 'ecr-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-amd64
                                """
                            }
                        },
                        "Push ARM64": {
                            withCredentials([usernamePassword(credentialsId: 'ecr-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh """
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                                docker push ${ECR_REPO}:${BUILD_NUMBER}-arm64
                                """
                            }
                        }
                    )
                }
            }
        }
    }
}
