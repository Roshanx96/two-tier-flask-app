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
                script {
                    /* 1. LOAD THE TOOL
                       We use the specific name 'OWASP-Dependency-Check' that you configured
                       in Manage Jenkins > Tools. This sets the path to the executable.
                    */
                    def depCheckHome = tool 'OWASP-Dependency-Check'
                    
                    /* 2. RUN THE SCAN
                       - --project flask-app: Sets the name of the project in the report.
                       - --scan .: Scans the current directory.
                       - --format ALL: Generates XML (required for Jenkins Sidebar) AND HTML (for download).
                       - (Removed --noupdate): Allows the tool to download the vulnerability database.
                    */
                    sh "${depCheckHome}/bin/dependency-check.sh --project flask-app --scan . --format ALL"
                }
            }
            post {
                always {
                    /* 3. PUBLISH TO SIDEBAR
                       This reads the 'dependency-check-report.xml' file generated above
                       and creates the "Dependency-Check" link/graph on the Jenkins sidebar.
                    */
                    dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    
                    /* 4. ARCHIVE ARTIFACTS
                       This saves the 'dependency-check-report.html' file so you can 
                       download it manually from the "Build Artifacts" section.
                    */
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dependency-check-report.html'
                }
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
