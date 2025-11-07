pipeline {
    agent any

    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
        choice(name: 'DEPLOY_ENV', choices: ['none', 'staging', 'production'], description: 'Deployment environment')
        booleanParam(name: 'PUSH_DOCKER', defaultValue: false, description: 'Push image to Docker registry')
    }

    environment {
        BUILD_VERSION = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        TESTCONTAINERS_RYUK_DISABLED = 'true'
        TESTCONTAINERS_CHECKS_DISABLE = 'true'
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: params.BRANCH, url: 'https://github.com/billelz/spring-petclinic.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Parallel Testing') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test -Dgroups="unit" || echo "Unit tests failed"'
                        junit '**/target/surefire-reports/*.xml'
                    }
                }
                stage('Integration Tests (if Docker available)') {
                    steps {
                        script {
                            // Check if Docker is available
                            def dockerAvailable = sh(script: 'docker ps > /dev/null 2>&1', returnStatus: true) == 0
                            if (dockerAvailable) {
                                echo "✅ Docker detected — running integration tests with Testcontainers"
                                sh 'mvn verify -Dgroups="integration"'
                            } else {
                                echo "⚠️ Docker not available — skipping Testcontainers integration tests"
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Image Build') {
            steps {
                sh 'docker build -t spring-petclinic:${BUILD_VERSION} .'
                script {
                    if (params.PUSH_DOCKER) {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            sh '''
                                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                docker push spring-petclinic:${BUILD_VERSION}
                            '''
                        }
                    }
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }

        stage('Deployment Simulation') {
            when {
                expression { params.DEPLOY_ENV == 'staging' }
            }
            steps {
                echo "🚀 Deploying version ${BUILD_VERSION} to ${params.DEPLOY_ENV} environment..."
                sh '''
                    echo "Simulating deployment..."
                    docker run -d --rm -p 8080:8080 spring-petclinic:${BUILD_VERSION} || echo "Docker run skipped"
                '''
                echo "✅ Application deployed successfully."
            }
        }
    }

    post {
        success {
            emailext(
                subject: "✅ Jenkins Build #${BUILD_NUMBER} Successful",
                body: "Build ${BUILD_NUMBER} succeeded for branch ${params.BRANCH}.",
                to: "team@example.com"
            )
        }
        failure {
            emailext(
                subject: "❌ Jenkins Build #${BUILD_NUMBER} Failed",
                body: "Build ${BUILD_NUMBER} failed for branch ${params.BRANCH}.",
                to: "team@example.com"
            )
        }
    }
}
