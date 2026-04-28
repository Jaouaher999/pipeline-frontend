pipeline {
    agent { label 'docker-agent' }

    environment {
        DOCKER_IMAGE = "yellowstar999/frontend"
        DOCKER_TAG   = "${env.BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "pipeline-frontend"                 
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES') 
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Install & Build') {
            steps {
                sh 'node --version && npm --version'
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${tool 'SonarScanner'}/bin/sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
                
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Trigger CD') {
            steps {
                build job: 'frontend-cd-pipeline', wait: false,
                      parameters: [string(name: 'IMAGE_TAG', value: "${DOCKER_TAG}")]
            }
        }
    }
    post { always { sh 'docker image prune -f' } }
}