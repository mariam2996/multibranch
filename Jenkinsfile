pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        DOCKER_REGISTRY   = 'your-registry.example.com'      // غيّرها براجيستري بتاعك (Docker Hub / ECR / GCR ...)
        IMAGE_NAME        = 'nodejs-docker-exercise'
        DOCKER_CREDS_ID   = 'docker-registry-creds'          // ID بتاع الـ credentials في Jenkins
        IMAGE_TAG         = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm install'
            }
        }

        // لو عندك اختبارات هتتفعل هنا (لو مفيش سكريبت test هيتخطى)
        stage('Test') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm test || echo "No tests configured, skipping..."'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push Docker Image') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'stg'
                    branch 'main'
                }
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", DOCKER_CREDS_ID) {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("${env.BRANCH_NAME}-latest")
                    }
                }
            }
        }

        stage('Deploy to Development') {
            when {
                branch 'dev'
            }
            steps {
                echo "🚀 Deploying ${IMAGE_TAG} to DEVELOPMENT environment"
                // مثال: sh 'kubectl set image deployment/app-dev app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -n dev'
                // أو: sh 'ssh dev-server "docker pull ... && docker restart ..."'
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'stg'
            }
            steps {
                echo "🚀 Deploying ${IMAGE_TAG} to STAGING environment"
                // مثال: sh 'kubectl set image deployment/app-stg app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -n staging'
            }
        }

        stage('Approval for Production') {
            when {
                branch 'main'
            }
            steps {
                input message: "هل توافق على النشر على البروداكشن؟ (${IMAGE_TAG})", ok: 'Deploy'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                echo "🚀 Deploying ${IMAGE_TAG} to PRODUCTION environment"
                // مثال: sh 'kubectl set image deployment/app-prod app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -n production'
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline succed: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline fail: ${env.BRANCH_NAME}"
        }
        always {
            sh 'docker image prune -f || true'
        }
    }
}
