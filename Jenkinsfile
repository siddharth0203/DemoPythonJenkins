pipeline {
  agent any

  environment {
    // 🔧 Jenkins credentials ID for Docker Hub
    DOCKERHUB_CREDS = credentials('dockerhub-password')

    // 🔧 Docker Hub image name (your repo)
    IMAGE_NAME = "siddharthjamalpur/siddharth-test"
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {

    stage('Checkout') {
      steps {
        echo "📦 Cloning repository..."
        checkout scm
        sh 'git branch -v || true'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          echo "🐳 Building Docker image..."
          sh """
            docker build -t generic-app:${BUILD_NUMBER} .
            docker tag generic-app:${BUILD_NUMBER} ${IMAGE_NAME}:${BUILD_NUMBER}
          """
        }
      }
    }

    stage('Scan Docker Image (Trivy)') {
      steps {
        script {
          echo "🔍 Scanning Docker image with Trivy..."
          // Scan the image for vulnerabilities and save report
          sh '''
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image \
              --exit-code 0 --no-progress --format json generic-app:${BUILD_NUMBER} > scan.json || true
          '''

          // Count vulnerabilities (using jq)
          def vulnCount = sh(script: "cat scan.json | jq '.Results[0].Vulnerabilities | length' 2>/dev/null || echo 0", returnStdout: true).trim()
          echo "🧮 Vulnerabilities found: ${vulnCount}"
          env.VULN_COUNT = vulnCount
        }
      }
    }

    stage('Evaluate Scan Results') {
      steps {
        script {
          def score = (env.VULN_COUNT ?: '0').toInteger()
          if (score >= 5) {
            echo "❌ Too many vulnerabilities (${score} >= 5). Aborting push."
            currentBuild.result = 'ABORTED'
            error("Build stopped due to high vulnerability count.")
          } else {
            echo "✅ Vulnerability score acceptable (${score} < 5). Proceeding to push."
          }
        }
      }
    }

    stage('Push to Docker Hub') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}", usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push ${IMAGE_NAME}:${BUILD_NUMBER}
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline completed with status: ${currentBuild.result ?: 'SUCCESS'}"
      archiveArtifacts artifacts: 'scan.json', allowEmptyArchive: true
      sh 'docker image rm generic-app:${BUILD_NUMBER} || true'
      sh "docker image rm ${IMAGE_NAME}:${BUILD_NUMBER} || true"
    }
    success {
      echo "🎉 Image pushed successfully to Docker Hub!"
    }
    aborted {
      echo "⚠️ Pipeline aborted due to vulnerabilities."
    }
    failure {
      echo "❌ Pipeline failed."
    }
  }
}
