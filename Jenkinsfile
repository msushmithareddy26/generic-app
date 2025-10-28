pipeline {
    agent any

    // Parameter for hotfix commit hash
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
                echo "Cloning repo and fetching all branches"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CloneOption', shallow: false, depth: 0, noTags: false]],
                    userRemoteConfigs: [[url: 'https://github.com/msushmithareddy26/generic-app.git']]
                ])
                sh 'git fetch origin hotfix:hotfix || true' // Ensure hotfix branch exists locally
            }
        }

        stage('Git Cherry-Pick Ops') {
            steps {
                script {
                    try {
                        sh "git checkout main"
                        sh "git cherry-pick ${params.COMMIT_HASH}"
                    } catch (err) {
                        // Log commit info and abort pipeline
                        sh "git log -1 --oneline ${params.COMMIT_HASH} || echo 'Commit not found'"
                        echo "Cherry-pick Error: ${err.getMessage()}"
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
                    try {
                        parallel(
                            "AMD64 Build": {
                                sh "docker buildx build --platform linux/amd64 -t ${ECR_REPO}:${BUILD_NUMBER}-amd64 --load ."
                            },
                            "ARM64 Build": {
                                sh "docker buildx build --platform linux/arm64 -t ${ECR_REPO}:${BUILD_NUMBER}-arm64 --load ."
                            }
                        )
                    } catch (err) {
                        echo "Parallel Build Error in ${env.STAGE_NAME}: ${err.getMessage()}"
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
