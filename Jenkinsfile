pipeline {
    agent any

    environment {
        VENV_PATH = 'venv'
        FLASK_APP = 'workspace/flask/app.py'
        PATH = "$VENV_PATH/Scripts:$PATH"
        SONARQUBE_SCANNER_HOME = tool name: 'SonarQube Scanner'
        SONARQUBE_TOKEN = 'squ_6b146edc8eca9957203e58ba7d82dfc1ffa52924'
        DEPENDENCY_CHECK_HOME = '/var/jenkins_home/tools/org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation/OWASP_Dependency-Check/dependency-check'
        PYTHON_PATH = 'C:\\Users\\wenwe\\AppData\\Local\\Microsoft\\WindowsApps\\python3.exe'
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
                    git branch: 'main', url: 'https://github.com/TPeiWen/test.git'
                }
            }
        }

        stage('Setup Virtual Environment') {
            steps {
                dir('workspace/flask') {
                    bat "${env.PYTHON_PATH} -m venv $VENV_PATH"
                }
            }
        }

        stage('Activate Virtual Environment and Install Dependencies') {
            steps {
                dir('workspace/flask') {
                    bat ".\\$VENV_PATH\\Scripts\\activate && pip install -r requirements.txt"
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
                    "%DEPENDENCY_CHECK_HOME%\\bin\\dependency-check.bat" --project "Flask App" --scan . --format "ALL" --out workspace\\flask\\dependency-check-report || true
                    '''
                }
            }
        }

        stage('UI Testing') {
            steps {
                script {
                    // Start the Flask app in the background
                    bat ".\\$VENV_PATH\\Scripts\\activate && set FLASK_APP=%FLASK_APP% && start flask run"
                    // Give the server a moment to start
                    bat 'timeout /t 5'
                    // Debugging: Check if the Flask app is running
                    bat 'curl -s http://127.0.0.1:5000 || echo "Flask app did not start"'

                    // Test a strong password
                    bat '''
                    curl -s -X POST -F "password=StrongPass123" http://127.0.0.1:5000 | findstr "Welcome"
                    '''

                    // Test a weak password
                    bat '''
                    curl -s -X POST -F "password=password" http://127.0.0.1:5000 | findstr "Password does not meet the requirements"
                    '''

                    // Stop the Flask app
                    bat 'taskkill /F /IM flask.exe'
                }
            }
        }

        stage('Integration Testing') {
            steps {
                dir('workspace/flask') {
                    bat ".\\$VENV_PATH\\Scripts\\activate && pytest --junitxml=integration-test-results.xml"
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
                        "%SONARQUBE_SCANNER_HOME%\\bin\\sonar-scanner" ^
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
                    bat 'for /F "tokens=*" %i in (\'docker ps --filter publish=5000 --format "{{.ID}}"\') do docker stop %i'
                    // Remove the stopped container
                    bat 'for /F "tokens=*" %i in (\'docker ps -a --filter status=exited --filter publish=5000 --format "{{.ID}}"\') do docker rm %i'
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
