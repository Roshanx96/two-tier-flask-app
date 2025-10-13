pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'Github-cred'
        DOCKER_CREDENTIALS = 'Dockerhub-cred'   
        DOCKERHUB_USERNAME = 'roshanx' // Your DockerHub username
        GITHUB_REPO = 'https://github.com/Roshanx96/two-tier-flask-app.git'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git credentialsId: "${env.GITHUB_CREDENTIALS}", url: "${env.GITHUB_REPO}"
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Filesystem Scan') {
                    steps {
                        sh 'trivy fs --exit-code 1 --severity CRITICAL --no-progress . || true'
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        sh 'dependency-check.sh --project flask-app --scan . || true'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONARQUBE_SERVER = 'SonarQube' // Jenkins SonarQube server name
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner'
                }
            }
        }

        stage('Wait for SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${env.DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                sh '''
                    docker build -t $DOCKERHUB_USERNAME/flask-app:latest .
                    docker push $DOCKERHUB_USERNAME/flask-app:latest
                '''
            }
        }
    }

    post {
        failure {
           echo "❌ Flask-app CI Pipeline Failed"
        }
    }
}
