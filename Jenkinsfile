pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "praveenkumar446/django-image"
        SCANNER_HOME = tool 'SonarScanner'
        VENV_DIR = ".venv"
    }

    stages {

        /* ---------------------------
            Checkout Source Code
        ----------------------------*/
        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                git branch: 'master', url: 'https://github.com/Praveenchenu/My_Store.git'
            }
        }

        /* ---------------------------
            Python Virtual Environment
        ----------------------------*/
        stage('Setup Python Virtual Environment') {
            steps {
                sh '''
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                '''
            }
        }

        /* ---------------------------
            Linting - flake8 (Install)
        ----------------------------*/
        stage('Install Linting Tools') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install flake8 black autoflake
                '''
            }
        }

        stage('Auto-fix Lint Issues') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate

                    pip install isort

                    isort .
                    autoflake --in-place --remove-unused-variables --remove-all-unused-imports -r .
                    black . --line-length 120
                '''
            }
        }

        stage('Run Flake8') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    flake8 --max-line-length=120 --exclude=${VENV_DIR} || true
                '''
            }
        }

        /* ---------------------------
            pip-audit SCA scan
        ----------------------------*/
        stage('SCA - pip-audit') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install pip-audit
                    pip-audit || true
                '''
            }
        }

        /* ---------------------------
            SonarQube Scan
        ----------------------------*/
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=My_Store \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://18.141.221.240:9000 \
                            -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        /* ---------------------------
            Install Dependencies
        ----------------------------*/
        stage('Install Dependencies') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install -r requirements.txt
                '''
            }
        }

        /* ---------------------------
            Unit Tests
        ----------------------------*/
        stage('Unit Tests with Coverage') {
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

        /* ---------------------------
            Build Docker Image
        ----------------------------*/
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
            }
        }

        /* ---------------------------
            Scan Docker Image
        ----------------------------*/
        stage('Scan Docker Image') {
            steps {
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        > trivyresults.txt || true

                    echo "Trivy scan completed. See trivyresults.txt"
                '''
            }
        }

        /* ---------------------------
            Push Docker Image
        ----------------------------*/
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-reistry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {

                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    '''
                }
            }
        }

        /* ---------------------------
            Update deployment.yaml
        ----------------------------*/
        stage('Update Deployment Files') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-credentials',usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        git config --global user.email "praveenchenu@gmail.com"
                        git config --global user.name "praveen"

                        sed -i "s|image:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|" deployment.yaml

                        git add deployment.yaml
                        git commit -m "Update image tag to ${BUILD_NUMBER}" || echo "No changes"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/Praveenchenu/My_Store.git HEAD:master
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed! Check logs."
        }
    }
}
