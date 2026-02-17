pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'Github-cred'
        DOCKER_CREDENTIALS = 'Dockerhub-cred'   
        DOCKERHUB_USERNAME = 'roshanx' 
        GITHUB_REPO = 'https://github.com/Roshanx96/two-tier-flask-app.git'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: "${env.GITHUB_CREDENTIALS}", url: "${env.GITHUB_REPO}"
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Filesystem Scan') {
                    steps {
                        sh 'trivy fs . --exit-code 1 --severity HIGH,CRITICAL || true'
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        // Ensure dependency-check is also installed or configured in tools if this fails too
                        sh 'dependency-check.sh --project flask-app --scan . || true'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    /* 1. TOOL DEFINITION
                       We manually define the scanner home because the 'tools {}' block 
                       doesn't work with some plugin versions. 
                       'SonarScanner' must match the Name in Manage Jenkins > Tools.
                    */
                    def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    
                    /* 2. SERVER CONNECTION
                       withSonarQubeEnv('SonarQube'):
                       - 'SonarQube' matches the Name in Manage Jenkins > System.
                       - This automatically injects:
                            $SONAR_HOST_URL (Server URL)
                            $SONAR_AUTH_TOKEN (Login Token)
                       - You DO NOT need to pass -Dsonar.host.url manually.
                    */
                    withSonarQubeEnv('SonarQube') {
                        
                        /* 3. THE SCAN COMMAND
                           -Dsonar.projectKey:  Unique ID for this project in SonarQube (Required).
                           -Dsonar.projectName: Friendly name displayed on the dashboard (Optional).
                           -Dsonar.sources:     Where the code is located. '.' means current directory (Required).
                        */
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=flask-app \
                        -Dsonar.projectName='Flask Application' \
                        -Dsonar.sources=.
                        """
                    }
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
        stage('Run Docker Containers'){
            steps{
                sh '''
                    docker-compose down || true 
                    docker-compose up -d --build 
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
