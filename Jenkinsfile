pipeline {
    agent any

    parameters {
        string(name: 'ECR_REPO_NAME', defaultValue: 'django-project', description: 'Enter ECR repository name')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '123456789012', description: 'Enter AWS Account ID')
    }

    environment {
        SCANNER_HOME = tool 'SonarQube Scanner'     // Update this name
        VENV_DIR = "${WORKSPACE}/venv"
    }

    stages {

        stage('1. Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Praveenchenu/My_Store.git'
            }
        }

        stage('2. Setup Python Virtual Environment') {
            steps {
                sh '''
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip setuptools wheel
                '''
            }
        }

        stage('3. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube Server') {   // Update this name
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=My_Store \
                                -Dsonar.projectName=My_Store \
                                -Dsonar.sources=. \
                                -Dsonar.python.version=3 \
                                -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('4. Install Dependencies') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install -r requirements.txt
                '''
            }
        }

        stage('5. Quality Gate Check') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('6. Unit Tests with Coverage') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install coverage
                    coverage run manage.py test
                    coverage report
                    coverage xml
                '''
            }
        }

        stage('7. SAST - Bandit Scan') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install bandit

                    mkdir -p ${WORKSPACE}/reports

                    bandit -r . \
                        --exclude ${VENV_DIR},migrations,__pycache__ \
                        -ll -f txt \
                        -o ${WORKSPACE}/reports/bandit_vulnerabilities.txt || true

                    echo "Bandit scan completed."
                '''
            }
        }

        stage('8. Trivy Scan') {
            steps {
                sh '''
                    trivy fs . > trivy-report.txt || true
                    echo "Trivy scan completed."
                '''
            }
        }

        stage('9. Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${ECR_REPO_NAME} .
                '''
            }
        }

        stage('10. Create ECR Repo') {
            steps {
                withCredentials([
                    string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')
         ]) {
                sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_KEY
                    aws ecr describe-repositories --repository-names ${ECR_REPO_NAME} --region us-east-1 || \
                    aws ecr create-repository --repository-name ${ECR_REPO_NAME} --region us-east-1
                '''
           }
        }
    }

        stage('11. Login & Tag Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')
        ]) {
                sh '''
                    set -e
                        ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${ECR_REPO_NAME}"

                        # Login to ECR
                        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_URI"

                        # Tag images
                        docker tag ${ECR_REPO_NAME}:latest "$ECR_URI:latest"
                        docker tag ${ECR_REPO_NAME}:latest "$ECR_URI:${BUILD_NUMBER}"
                '''
           }
        }
    }

        stage('12. Push Image') {
            steps {
                sh '''
                    set -e
                        ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${ECR_REPO_NAME}"
                        docker push "$ECR_URI:latest"
                        docker push "$ECR_URI:${BUILD_NUMBER}"
                '''
            }
        }

        stage('13. Cleanup') {
            steps {
                sh '''
                    set -e
                        ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${ECR_REPO_NAME}"
                        docker rmi "$ECR_URI:latest" || true
                        docker rmi "$ECR_URI:${BUILD_NUMBER}" || true
                        docker rmi ${ECR_REPO_NAME} || true
                        docker image prune -f
                '''
            }
        }
