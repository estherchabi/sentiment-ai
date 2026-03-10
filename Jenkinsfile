// Jenkinsfile Pipeline CI/CD SentimentAI
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'sentiment-ai'
        REGISTRY   = 'ghcr.io/estherchabi'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
                echo "Branche : ${env.BRANCH_NAME}"
                echo "Commit  : ${env.GIT_COMMIT}"
                echo "Tag     : ${env.IMAGE_TAG}"
                sh 'git log --oneline -5'
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    docker run --rm \
                        -v "$(pwd):/app" \
                        -w /app \
                        python:3.11-slim \
                        sh -c 'pip install flake8 -q && flake8 src/ --max-line-length=100'
                '''
            }
        }

        stage('Build & Test') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."

                sh """
                    docker run --rm \
                        -e CI=true \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        pytest tests/ -v \
                        --cov=src \
                        --cov-report=term-missing \
                        --cov-fail-under=70
                """
            }
            post {
                failure {
                    echo 'Tests échoués ou couverture insuffisante (< 70%)'
                }
            }
        }

        stage('Push') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh """
                        echo "\$REGISTRY_PASS" | docker login ghcr.io \
                            -u "\$REGISTRY_USER" --password-stdin

                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:main

                        docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${REGISTRY}/${IMAGE_NAME}:main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline réussi ! Image : ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline échoué. Consultez les logs ci-dessus.'
        }
        always {
            sh 'docker compose down -v 2>/dev/null || true'
        }
    }
}