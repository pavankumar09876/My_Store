pipeline {
    agent any
    tools {
        maven 'maven'
    }

    stages {
        stage ('checkout') {
            steps {
                echo 'Checking out source code...'
                git branch: 'master', url: 'https://github.com/Praveenchenu/My_Store.git'
            }
        }

        stage ('build') {
            steps {
                echo 'Building the project...'
                sh 'mvn clean install'
            }
        }

        stage ('sonarQube') {
            steps {
                sh 'ls -ltr'
                sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=sonar-token \
                    -Dsonar.host.url=http://3.1.196.239:9000 \
                    -Dsonar.login=sqa_bbadc1a60d2a2ec37ba1dbbc31bcd7598a548d3e
                '''
            }
        }

        stage ('build artifacts') {
            steps {
                echo 'Building artifacts...'
                sh 'mvn package'
            }
        }

        stage ('Docker Build') {
            steps {
                echo 'Building Docker image...'
                sh "docker build -t praveenkumar446/django-image:latest:${env.BUILD_NUMBER} ."
            }
        }

        stage ('push to registry') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-hub-credentials', variable: 'DOCKER_HUB_PASSWORD')]) {
                        // NOTE: Corrected variable name from 'dockerhub' to 'DOCKER_HUB_PASSWORD' for clarity in the sh block
                        sh '''
                            echo $DOCKER_HUB_PASSWORD | docker login -u praveenkumar446 --password-stdin
                            docker push praveenkumar446/django-image:latest:${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }

        stage ('update deployment files') {
            steps {
                echo 'Updating deployment files...'
                // NOTE: The withCredentials is not strictly needed here if the credential variable isn't used in the sed command.
                // However, if the stage structure was intended to be separate, it's safer to keep the credential scope narrow.
                sh "sed -i 's|image: praveenkumar446/django-image:.*|image: praveenkumar446/django-image:latest:${env.BUILD_NUMBER}|' k8s/deployment.yaml"
            }
        }
    }
}
