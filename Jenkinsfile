pipeline {
    agent any

    tools {
        jdk 'jdk21'
        maven 'maven'
    }

    environment {
        IMAGE_NAME = "petclinic-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Bertrussoff/spring-petclinic.git'
            }
        }

        stage('Build (Skip Tests)') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    mvn sonar:sonar \
                      -Dsonar.projectKey=petclinic \
                      -Dsonar.host.url=http://192.168.64.5:9000 \
                      -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        // ✅ NEW: Quality Gate (fail if Sonar fails)
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Check Build Output') {
            steps {
                sh 'ls -ltr target/'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        // ✅ UPDATED: Trivy FS scan with fail condition
        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs \
                  --cache-dir /tmp/trivy-cache \
                  --severity HIGH,CRITICAL \
                  --exit-code 1 \
                  --no-progress \
                  .
                '''
            }
        }

        // ✅ UPDATED: Trivy Image scan with fail condition
        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image \
                  --cache-dir /tmp/trivy-cache \
                  --severity HIGH,CRITICAL \
                  --exit-code 1 \
                  --no-progress \
                  $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        // ✅ MOVED: Cleanup at the end (best practice)
        stage('Cleanup') {
            steps {
                sh 'docker system prune -f'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully (SECURE)'
        }
        failure {
            echo '❌ Pipeline failed due to code issues or vulnerabilities'
        }
    }
}
