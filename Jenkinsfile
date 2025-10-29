pipeline {
    agent any

    environment {
        // 👇 Replace these with your details
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-password') // Jenkins credential ID
        IMAGE_NAME = "siddharthjamalpur/siddharth-test"                         // Docker Hub repo name
        GIT_REPO = "https://github.com/siddharth0203/DemoPythonJenkins.git"    // Your GitHub repo URL
    }

    stages {

        stage('Clone Repository') {
            steps {
                echo "Cloning the repository..."
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building the Docker image..."
                // 👇 Change path if your Dockerfile is not in the root folder
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                echo "Logging in to Docker Hub..."
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
            }
        }
        
        stage('Scan Image Locally with Grype') {
          steps {
            echo "🔍 Scanning Docker image with Anchore Grype..."
            sh """
              docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                anchore/grype:latest ${IMAGE_NAME}:latest
            """
          }
        }
        
        stage('Check Vulnerability Score') {
            steps {
                script {
                    def highCriticalCount = sh(script: "cat grype-report.json | jq '.matches[].vulnerability.severity' | grep -E 'High|Critical' | wc -l", returnStdout: true).trim()
                    echo "Number of High/Critical vulnerabilities: ${highCriticalCount}"

                    if (highCriticalCount.toInteger() > 0) {
                        error("❌ Vulnerabilities too high — stopping pipeline.")
                    } else {
                        echo "✅ Vulnerability level acceptable — proceeding to push."
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "Pushing the Docker image to Docker Hub..."
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
    }

    post {
        success {
            echo "✅ Build and Push successful!"
        }
        failure {
            echo "❌ Build failed. Check logs."
        }
    }
}
