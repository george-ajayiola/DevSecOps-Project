pipeline {
    agent {
        label 'jenkins-agent'
    }
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "netflix-clone"
        RELEASE = "1.0.0"
        DOCKER_USER = "georgeao"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        DOCKER_REGISTRY_CREDENTIALS = 'docker' // Credentials ID for Docker registry
        NVD_API_KEY = credentials('nvd-api-key') // Use the credentials ID for NVD API key
    }
    stages {
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/george-ajayiola/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix-clone -Dsonar.projectKey=Netflix-clone"
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: DOCKER_REGISTRY_CREDENTIALS, toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=93c063cd8adfe5ac8add285ecc465dfe -t ${IMAGE_NAME} ."
                        sh "docker tag ${IMAGE_NAME} ${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > trivyimage.txt"
            }
        }
        stage('Deploy to container') {
            steps {
                sh "docker run -d -p 8081:80 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }
    post {
        always {
            emailext (
                subject: "Build Status: ${currentBuild.currentResult}",
                body: """Build Details:
                         - Job Name: ${env.JOB_NAME}
                         - Build Number: ${env.BUILD_NUMBER}
                         - Build Status: ${currentBuild.currentResult}
                         - Build URL: ${env.BUILD_URL}

                         Attached are the Trivy scan results for filesystem and Docker image.""",
                to: 'your-email@example.com',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt'
            )
        }
    }
}
