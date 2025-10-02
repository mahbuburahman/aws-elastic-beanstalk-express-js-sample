pipeline {
    agent {
        docker {
            image 'node:16'   // Build agent
            args '-u root:root' // run as root for npm install/docker if needed
        }
    }

    environment {
        REGISTRY       = credentials('docker-registry-creds')   // Jenkins credential (username+password or token)
        REGISTRY_URL   = "docker.io"                            // change if using GHCR/ECR
        REGISTRY_NS    = "your-docker-namespace"                // e.g. your DockerHub username
        IMAGE_NAME     = "nodejs-sample"
        SNYK_TOKEN     = credentials('snyk-token')              // Jenkins secret text credential
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'    // deterministic installs
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'npm test || true'  // allow npm test, adjust if tests are in repo
            }
            post {
                unsuccessful {
                    error("Unit tests failed ❌")
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${REGISTRY_NS}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh """
                        echo "Building image: $imageTag"
                        docker build -t $imageTag .
                    """
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh """
                    echo "Running Snyk scan..."
                    npm install -g snyk
                    snyk auth ${SNYK_TOKEN}
                    snyk test --severity-threshold=high || { echo "High/Critical vulnerabilities found"; exit 1; }
                """
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    def imageTag = "${REGISTRY_NS}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh """
                        echo $REGISTRY_PSW | docker login -u $REGISTRY_USR --password-stdin $REGISTRY_URL
                        docker push $imageTag
                    """
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/npm-debug.log', allowEmptyArchive: true
        }
        failure {
            echo 'Build failed ❌'
        }
        success {
            echo 'Build succeeded ✅'
        }
    }
}
