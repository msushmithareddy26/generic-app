pipeline {
    agent any

    environment {
        AWS_REGION = "eu-north-1"
        AWS_ACCOUNT_ID = "527930216402"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/generic-repo"
        IMAGE_TAG_AMD64 = "${ECR_REPO}:${BUILD_NUMBER}-amd64"
        IMAGE_TAG_ARM64 = "${ECR_REPO}:${BUILD_NUMBER}-arm64"
    }

    parameters {
        string(name: 'COMMIT_HASH', defaultValue: '', description: 'Hotfix commit hash')
    }

    stages {

        // --------------------------
        stage('Checkout') {
            steps {
                echo "Using Jenkins SCM checkout (no manual git clone required)"
            }
        }

        // --------------------------
        stage('Cherry-Pick Hotfix') {
            steps {
                script {
                    if (params.COMMIT_HASH?.trim()) {
                        sh 'git config user.email "m.sushmithareddy26@gmail.com"'
                        sh 'git config user.name "msushmithareddy26"'

                        echo "Cherry-picking commit: ${params.COMMIT_HASH}"
                        sh "git fetch origin hotfix"
                        sh "git checkout main"

                        try {
                            def status = sh(script: "git cherry-pick ${params.COMMIT_HASH} || true", returnStatus: true)

                            if (status == 0) {
                                echo "Cherry-pick applied successfully."
                            } else {
                                def output = sh(script: "git status --porcelain", returnStdout: true).trim()
                                if (output == "") {
                                    echo "Cherry-pick already applied (empty commit). Skipping."
                                    sh "git cherry-pick --abort || true"
                                } else {
                                    def commitDetails = sh(script: "git log -1 --oneline ${params.COMMIT_HASH}", returnStdout: true).trim()
                                    echo "Cherry-pick Error: Commit could not be applied\nCommit: ${commitDetails}"
                                    currentBuild.result = 'ABORTED'
                                    error("Aborting due to cherry-pick failure")
                                }
                            }
                        } catch (err) {
                            def commitDetails = sh(script: "git log -1 --oneline ${params.COMMIT_HASH}", returnStdout: true).trim()
                            echo "Cherry-pick Exception: ${err}\nCommit: ${commitDetails}"
                            currentBuild.result = 'ABORTED'
                            error("Aborting due to cherry-pick exception")
                        }

                    } else {
                        echo "No hotfix commit provided, skipping cherry-pick."
                    }
                }
            }
        }

        // --------------------------
        stage('Build Multi-Arch Images') {
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

        // --------------------------
        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'ecr-creds']]) {
                    script {
                        sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker push ${IMAGE_TAG_AMD64}
                        docker push ${IMAGE_TAG_ARM64}
                        """
                    }
                }
            }
        }

    } 
}
