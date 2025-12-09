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
           Linting - flake8
        ----------------------------*/
        stage('Linting - flake8') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install flake8
                    flake8 --max-line-length=120 --exclude=.venv
                '''
            }
        }

        /* ---------------------------
           Bandit Security Scan
        ----------------------------*/
        stage('SAST - Bandit Scan') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install bandit

                    # Create reports directory if it doesn't exist
                    mkdir -p reports

                    # Run Bandit scan on your source code and third-party libraries
                    bandit -r . --exclude ${VENV_DIR},migrations,__pycache__ -ll -f txt -o reports/bandit_vulnerabilities.txt || true

                    echo "Bandit scan completed. Vulnerabilities report saved as reports/bandit_vulnerabilities.txt"
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
                    pip-audit
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
                            -Dsonar.host.url=http://18.142.113.58:9000 \
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
           Run Django Unit Tests
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

        // ----------------------- SCAN DOCKER IMAGE -----------------------
        stage('Scan Docker Image') {
            steps {
                sh """
                    trivy image \
                        --scanners vuln \
                        --offline-scan \
                        ${registry}:latest \
                        > trivyresults.txt
                """
            }
        }

        /* ---------------------------
           Docker Push
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
           Update deployment.yaml in Repo
        ----------------------------*/
        stage('Update Deployment Files') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
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
