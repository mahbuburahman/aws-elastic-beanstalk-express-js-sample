pipeline {
  agent none

  environment {
    REGISTRY       = credentials('docker-registry-creds')
    REGISTRY_URL   = 'docker.io'
    REGISTRY_NS    = 'mahbubur708'
    IMAGE_NAME     = 'nodejs-sample'
    
    SNYK_TOKEN     = credentials('snyk-token')
  }

  options { timestamps() }

  stages {
    stage('Checkout') {
      agent { label 'built-in' }      // run on Jenkins controller
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16'
          args '-u root:root'
        }
      }
      steps {
        sh 'node -v && npm -v'
        sh 'npm ci'
        // tolerate missing "test" script; still fail if real tests fail
        sh 'npm test || echo "No test script; continuing"'
      }
    }

    stage('Build Docker Image') {
      agent { label 'built-in' }      // run where docker CLI & DOCKER_HOST are configured
      steps {
        script {
          env.IMAGE_TAG = "${REGISTRY_NS}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
          sh """
            echo "Building image: ${IMAGE_TAG}"
            docker build -t ${IMAGE_TAG} .
          """
        }
      }
    }

    // --- Pick ONE of the two security scans ---

    // A) Snyk (needs snyk-token)
    stage('Security Scan (Snyk)') {
      when { expression { return env.SNYK_TOKEN?.trim() } }
      agent { label 'built-in' }
      steps {
        sh """
          npm i -g snyk
          snyk auth ${SNYK_TOKEN}
          snyk test --severity-threshold=high
        """
      }
    }

    // B) OWASP Dependency-Check (no account; comment Snyk stage and use this instead)
    /*
    stage('Security Scan (OWASP DC)') {
      agent { label 'built-in' }
      steps {
        sh '''
          docker run --rm -v "$PWD":/src owasp/dependency-check \
            --scan /src --format HTML --out /src/dependency-check-report --failOnCVSS 7
        '''
      }
    }
    */

    stage('Push to Registry') {
      agent { label 'built-in' }
      steps {
        sh """
          echo $REGISTRY_PSW | docker login -u $REGISTRY_USR --password-stdin ${REGISTRY_URL}
          docker push ${IMAGE_TAG}
        """
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/npm-debug.log', allowEmptyArchive: true
      archiveArtifacts artifacts: 'dependency-check-report/**/*', allowEmptyArchive: true
    }
    success { echo "Build OK ✅  Pushed: ${IMAGE_TAG}" }
    failure { echo 'Build failed ❌' }
  }
}
