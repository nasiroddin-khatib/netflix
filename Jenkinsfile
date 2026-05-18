pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'

        AWS_ACCOUNT_ID = '245246852393'
        AWS_REGION = 'ap-south-1'

        IMAGE_NAME = 'netflix'
        IMAGE_TAG = 'latest'

        DOCKER_HUB = 'nasir590'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                url: 'https://github.com/nasiroddin-khatib/netflix'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build') {
            steps {
withCredentials([string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')]) {

    sh """
    $SCANNER_HOME/bin/sonar-scanner \
    -Dsonar.projectName=Netflix \
    -Dsonar.projectKey=Netflix \
    -Dsonar.login=$SONAR_TOKEN
    """
}            
}
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push $DOCKER_HUB/$IMAGE_NAME:$IMAGE_TAG'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image $DOCKER_HUB/$IMAGE_NAME:$IMAGE_TAG > trivyimage.txt'
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                dir('Kubernetes') {

                    withKubeConfig(
                        credentialsId: 'k8s',
                        restrictKubeConfigAccess: false
                    ) {

                        sh 'kubectl apply -f deployment.yml'
                        sh 'kubectl apply -f service.yml'
                    }
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
            subject: "${currentBuild.result} : ${env.JOB_NAME}",
            body: """
            Project: ${env.JOB_NAME}

            Build Number: ${env.BUILD_NUMBER}

            URL: ${env.BUILD_URL}
            """,
            to: 'nasirkhatib01@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
