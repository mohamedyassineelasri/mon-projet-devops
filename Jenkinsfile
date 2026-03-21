pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_BACKEND = 'monuser/laravel-backend'
        DOCKER_IMAGE_FRONTEND = 'monuser/react-frontend'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }
        
        stage('Test Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm test -- --passWithNoTests'
                }
            }
        }
        
        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'composer install --no-interaction'
                    sh 'cp .env.example .env'
                    sh 'php artisan key:generate'
                }
            }
        }
        
        stage('Test Backend') {
            steps {
                dir('backend') {
                    sh 'php artisan migrate --force'
                    sh 'php artisan test'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=mon-projet \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://localhost:9000 \
                          -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE_BACKEND}:${DOCKER_TAG}", "./backend")
                    docker.build("${DOCKER_IMAGE_FRONTEND}:${DOCKER_TAG}", "./frontend")
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        docker.image("${DOCKER_IMAGE_BACKEND}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE_BACKEND}:${DOCKER_TAG}").push('latest')
                        docker.image("${DOCKER_IMAGE_FRONTEND}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE_FRONTEND}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                    docker-compose down
                    docker-compose pull
                    docker-compose up -d
                '''
            }
        }
    }
    
    post {
        success {
            slackSend(
                color: 'good',
                message: "Pipeline réussi: ${env.JOB_NAME} - ${env.BUILD_NUMBER}\nConsulter les logs: ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Pipeline échoué: ${env.JOB_NAME} - ${env.BUILD_NUMBER}\nConsulter les logs: ${env.BUILD_URL}"
            )
        }
    }
}