pipeline {
    agent any

    environment {
        AWS_REGION = "eu-north-1"
        AWS_ACCOUNT_ID = "527930216402"
        ECR_REPO = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/generic-app"
        IMAGE_TAG_AMD64 = "generic-app:${BUILD_NUMBER}-amd64"
        IMAGE_TAG_ARM64 = "generic-app:${BUILD_NUMBER}-arm64"
        AWS_CREDENTIALS = credentials('ecr-creds')
    }

    parameters {
        string(name: 'COMMIT_HASH', defaultValue: '', description: 'Hotfix commit hash')
    }

    options {
        skipDefaultCheckout() // We'll do SCM checkout manually to avoid conflicts
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: 'main']],
                          doGenerateSubmoduleConfigurations: false,
                          userRemoteConfigs: [[url: 'https://github.com/msushmithareddy26/generic-app.git', credentialsId: 'github-cred']]
                ])
            }
        }

        stage('Cherry-Pick Hotfix') {
            steps {
                script {
                    if (params.COMMIT_HASH?.trim()) {
                        sh 'git config user.email "m.sushmithareddy26@gmail.com"'
                        sh 'git config user.name "msushmithareddy26"'
                        sh "git fetch origin hotfix"
                        sh "git checkout main"

                        try {
                            sh "git cherry-pick ${params.COMMIT_HASH}"
                            echo "Cherry-pick applied successfully."
                        } catch (err) {
                            // Log commit details for debugging
                            def commitDetails = sh(script: "git log -1 --oneline ${params.COMMIT_HASH}", returnStdout: true).trim()
                            echo "Cherry-pick Error: ${err}\nCommit: ${commitDetails}"
                            currentBuild.result = 'ABORTED'
                            error("Aborting due to cherry-pick failure")
                        }
                    } else {
                        echo "No hotfix commit provided, skipping cherry-pick."
                    }
                }
            }
        }

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
                                error("Aborting all parallel builds")
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
                                error("Aborting all parallel builds")
                            }
                        }
                    }
                }
            }
        }

        stage('Push to ECR') {
            when {
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'ecr-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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
