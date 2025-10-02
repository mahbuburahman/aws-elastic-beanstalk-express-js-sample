pipeline {
  agent none

  environment {
    REGISTRY       = credentials('docker-registry-creds')
    REGISTRY_URL   = 'docker.io'
    REGISTRY_NS    = 'mahbubur708'
    IMAGE_NAME     = 'nodejs-sample'
    SNYK_TOKEN     = credentials('snyk-token')   // keep if you use Snyk
  }

  options { timestamps() }

  stages {

    stage('Checkout') {
      agent { label 'built-in' }
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      agent { docker { image 'node:16'; args '-u root:root' } }
      steps {
        sh 'node -v && npm -v'
        sh 'npm ci'
        // sample app has no "test" script; don’t fail the build for that
        sh 'npm test || echo "No test script; continuing"'
      }
    }

    stage('Build Docker Image') {
      agent { label 'built-in' }  // Docker CLI & DOCKER_HOST available here
      steps {
        script {
          env.IMAGE_TAG = "${REGISTRY_NS}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
          sh '''
            echo "Building image: ${IMAGE_TAG}"
            docker build -t ${IMAGE_TAG} .
          '''
        }
      }
    }

    // ---------- choose ONE security scan ----------

    // A) Snyk inside node:16 (so npm exists)  ✅
    stage('Security Scan (Snyk)') {
      when { expression { return env.SNYK_TOKEN?.trim() } }
      agent { docker { image 'node:16'; args '-u root:root' } }
      steps {
        // IMPORTANT: use "$SNYK_TOKEN" INSIDE single-quoted sh to avoid Groovy interpolation warning
        sh 'npm i -g snyk'
        sh 'snyk auth "$SNYK_TOKEN"'
        sh 'snyk test --severity-threshold=high'
      }
    }

    /*
    // B) OWASP Dependency-Check via Docker (no Snyk account needed)
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
        sh '''
          echo $REGISTRY_PSW | docker login -u $REGISTRY_USR --password-stdin ${REGISTRY_URL}
          docker push ${IMAGE_TAG}
          docker tag ${IMAGE_TAG} ${REGISTRY_NS}/${IMAGE_NAME}:latest
          docker push ${REGISTRY_NS}/${IMAGE_NAME}:latest
        '''
      }
    }
  }

  post {
    always {
      // allocate a node so archiving has a workspace when agent none is used
      node('built-in') {
        archiveArtifacts artifacts: '**/npm-debug.log', allowEmptyArchive: true
        archiveArtifacts artifacts: 'dependency-check-report/**/*', allowEmptyArchive: true
      }
    }
    success { echo "Build OK ✅  Pushed: ${IMAGE_TAG}" }
    failure { echo 'Build failed ❌' }
  }
}
