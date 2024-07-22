pipeline {
    agent any

    environment {
        VENV_PATH = 'venv'
        FLASK_APP = 'workspace/flask/app.py'  // Correct path to the Flask app
        PATH = "${env.WORKSPACE}/${VENV_PATH}/Scripts;${env.PATH}"
        SONARQUBE_SCANNER_HOME = tool name: 'SonarQube Scanner'
        SONARQUBE_TOKEN = 'squ_c697e8812497d2bd59bc0201abc3f17ed5aa0488'  // Set your new SonarQube token here
        DEPENDENCY_CHECK_HOME = 'C:\\path\\to\\dependency-check'  // Update this path to the Dependency-Check installation directory
    }
    
    stages {
        stage('Check Docker') {
            steps {
                bat 'docker --version'
            }
        }
        
        stage('Clone Repository') {
            steps {
                dir('workspace') {
                    git branch: 'main', url: 'https://github.com/TPeiWen/test.git', credentialsId: 'a1685372-9a24-4431-8e36-81b4e39765e9'
                }
            }
        }
        
        stage('Setup Virtual Environment') {
            steps {
                dir('workspace/flask') {
                    bat 'python -m venv %VENV_PATH%'
                }
            }
        }
        
        stage('Activate Virtual Environment and Install Dependencies') {
            steps {
                dir('workspace/flask') {
                    bat '''
                    %VENV_PATH%\\Scripts\\activate && pip install -r requirements.txt
                    '''
                }
            }
        }
        
        stage('Dependency Check') {
            steps {
                script {
                    // Create the output directory for the dependency check report
                    bat 'mkdir workspace\\flask\\dependency-check-report'
                    // Print the dependency check home directory for debugging
                    bat 'echo Dependency Check Home: %DEPENDENCY_CHECK_HOME%'
                    bat 'dir %DEPENDENCY_CHECK_HOME%\\bin'
                    bat '''
                    %DEPENDENCY_CHECK_HOME%\\bin\\dependency-check.bat --project "Flask App" --scan . --format "ALL" --out workspace\\flask\\dependency-check-report || exit /b %ERRORLEVEL%
                    '''
                }
            }
        }
        
        stage('UI Testing') {
            steps {
                script {
                    // Start the Flask app in the background
                    bat '''
                    start /B %VENV_PATH%\\Scripts\\activate && set FLASK_APP=%FLASK_APP% && flask run
                    '''
                    // Give the server a moment to start
                    bat 'timeout /t 5'
                    // Debugging: Check if the Flask app is running
                    bat 'curl -s http://127.0.0.1:5000 || echo Flask app did not start'
                    
                    // Test a strong password
                    bat '''
                    curl -s -X POST -F "password=StrongPass123" http://127.0.0.1:5000 | findstr "Welcome"
                    '''
                    
                    // Test a weak password
                    bat '''
                    curl -s -X POST -F "password=password" http://127.0.0.1:5000 | findstr "Password does not meet the requirements"
                    '''
                    
                    // Stop the Flask app
                    bat 'taskkill /IM flask.exe /F'
                }
            }
        }
        
        stage('Integration Testing') {
            steps {
                dir('workspace/flask') {
                    bat '%VENV_PATH%\\Scripts\\activate && pytest --junitxml=integration-test-results.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                dir('workspace/flask') {
                    bat 'docker build -t flask-app .'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    dir('workspace/flask') {
                        bat '''
                        %SONARQUBE_SCANNER_HOME%\\bin\\sonar-scanner.bat ^
                        -Dsonar.projectKey=flask-app ^
                        -Dsonar.sources=. ^
                        -Dsonar.inclusions=app.py ^
                        -Dsonar.host.url=http://sonarqube:9000 ^
                        -Dsonar.login=%SONARQUBE_TOKEN%
                        '''
                    }
                }
            }
        }
        
        stage('Deploy Flask App') {
            steps {
                script {
                    echo 'Deploying Flask App...'
                    // Stop any running container on port 5000
                    bat 'docker ps --filter "publish=5000" --format "{{.ID}}" | xargs -r docker stop'
                    // Remove the stopped container
                    bat 'docker ps -a --filter "status=exited" --filter "publish=5000" --format "{{.ID}}" | xargs -r docker rm'
                    // Run the new Flask app container
                    bat 'docker run -d -p 5000:5000 flask-app'
                    bat 'timeout /t 10'
                }
            }
        }
    }
    
    post {
        failure {
            script {
                echo 'Build failed, not deploying Flask app.'
            }
        }
        always {
            archiveArtifacts artifacts: 'workspace/flask/dependency-check-report/*.*', allowEmptyArchive: true
            archiveArtifacts artifacts: 'workspace/flask/integration-test-results.xml', allowEmptyArchive: true
        }
    }
}
