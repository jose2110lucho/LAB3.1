pipeline {
  agent any

  environment {
    DOCKER_REGISTRY = "myregistry.example.com"
    DOCKER_CREDENTIALS = "docker-registry-credentials"
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "${env.DOCKER_REGISTRY}/devsecops-labs/app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://host.docker.internal:3000"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        bat 'dir'
      }
    }

    stage('SAST - Semgrep') {
      steps {
        echo "Running Semgrep (SAST)..."
        bat """
          docker run --rm -v "%CD%":/src returntocorp/semgrep:latest semgrep --config=auto /src/src --json > semgrep-results.json 2>nul
          type semgrep-results.json || ver>nul
        """
        archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
      }
      post {
        always {
          echo 'Semgrep done.'
        }
      }
    }

    stage('SCA - Dependency Check (OWASP dependency-check)') {
      steps {
        echo "Running SCA / Dependency-Check..."
        bat """
          if not exist dependency-check-reports mkdir dependency-check-reports
          docker run --rm -v "%CD%":/src -v odc_cache:/usr/share/dependency-check/data -e NVD_API_KEY=%NVD_API_KEY% owasp/dependency-check:latest dependency-check --project "devsecops-labs" --scan /src --format JSON --out /src/dependency-check-reports || ver>nul
        """
        archiveArtifacts artifacts: 'dependency-check-reports/**', allowEmptyArchive: true
      }
    }

    stage('Build') {
      steps {
        echo "Building app (npm install and tests)..."
        bat """
          docker run --rm -v "%CD%":/workspace -w /workspace/src node:18 bash -lc "\
            npm install --no-audit --no-fund && \
            (npm test --silent || echo 'Tests failed (continue)')\
          "
        """
      }
    }

    stage('Docker Build & Trivy Scan') {
      steps {
        echo "Building Docker image..."
        bat """
          docker build -t %DOCKER_IMAGE_NAME% -f Dockerfile .
        """
        echo "Scanning image with Trivy..."
        bat """
          if not exist trivy-reports mkdir trivy-reports
          docker run --rm -v //var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --format json --output trivy-reports/trivy-report.json %DOCKER_IMAGE_NAME% || ver>nul
          docker run --rm -v //var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL %DOCKER_IMAGE_NAME% || ver>nul
        """
        archiveArtifacts artifacts: 'trivy-reports/**', allowEmptyArchive: true
      }
    }

    stage('Push Image (optional)') {
      when {
        expression { return env.DOCKER_REGISTRY != null && env.DOCKER_REGISTRY != "" }
      }
      steps {
        echo "Pushing image to registry ${DOCKER_REGISTRY}..."
        withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          bat """
            echo %DOCKER_PASS% | docker login %DOCKER_REGISTRY% -u "%DOCKER_USER%" --password-stdin
            docker push %DOCKER_IMAGE_NAME%
            docker logout %DOCKER_REGISTRY%
          """
        }
      }
    }

    stage('Deploy to Staging (docker-compose)') {
      steps {
        echo "Deploying to staging with docker-compose..."
        bat """
          docker compose -f docker-compose.yml down || ver>nul
          docker compose -f docker-compose.yml up -d --build
          timeout /t 8 /nobreak >nul
          docker ps -a
        """
      }
    }

    stage('DAST - OWASP ZAP scan') {
      steps {
        echo "Running DAST (OWASP ZAP) against ${STAGING_URL} ..."
        bat """
          if not exist zap-reports mkdir zap-reports
          docker run --rm -v "%CD%":/zap/wrk:rw ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t %STAGING_URL% -r zap-reports/zap-report.html || ver>nul
        """
        archiveArtifacts artifacts: 'zap-reports/**', allowEmptyArchive: true
      }
    }

    stage('Policy Check - Fail on HIGH/CRITICAL CVEs') {
      when {
        expression { return isUnix() }
      }
      steps {
        sh '''
          chmod +x scripts/scan_trivy_fail.sh
          ./scripts/scan_trivy_fail.sh $DOCKER_IMAGE_NAME || exit_code=$?
          if [ "${exit_code:-0}" -eq 2 ]; then
            echo "Failing pipeline due to HIGH/CRITICAL vulnerabilities detected by Trivy."
            exit 1
          fi
        '''
      }
    }

  } // stages

  post {
    always {
      echo "Pipeline finished. Collecting artifacts..."
    }
    failure {
      echo "Pipeline failed!"
    }
  }
}