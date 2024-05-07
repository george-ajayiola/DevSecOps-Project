
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        APP_NAME="netflix-clone"
        RELEASE= "1.0.0"
        DOCKER_USER="georgeao"
        DOCKER_PASS="docker"
        IMAGE_NAME= "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG= "${RELEASE}-${BUILD_NUMBER}"
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
      
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
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
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }

        }
        
        stage("TRIVY"){
            steps{
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh "docker run -d -p 8081:80 ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
    }
}




