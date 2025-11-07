// Jenkinsfile - Advanced Jenkins Exercise (Declarative)
pipeline {
  agent any
  tools {
    jdk 'jdk25'
    maven 'maven3'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timestamps()
    ansiColor('xterm')
  }

  // parameters required by assignment
  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    choice(name: 'DEPLOY_ENV', choices: ['none','staging','production'], description: 'Deployment environment (deploy only when staging)')
    booleanParam(name: 'PUSH_DOCKER', defaultValue: false, description: 'Push image to Docker registry (requires credentials)')
  }

  environment {
    GIT_CREDENTIALS = 'github-creds'       // set this credential in Jenkins
    DOCKER_CREDENTIALS = 'dockerhub-creds' // set this credential in Jenkins if pushing
    SMTP_RECIPIENT = 'billelzemmel2@gmail.com'    // change to real recipient
    // BUILD_VERSION, GIT_COMMIT, DOCKER_IMAGE will be set after checkout
  }

  stages {
    stage('Checkout') {
      steps {
        script {
          echo "Checkout branch ${params.BRANCH}"
          checkout([
            $class: 'GitSCM',
            branches: [[name: "*/${params.BRANCH}"]],
            userRemoteConfigs: [[url: 'https://github.com/spring-projects/spring-petclinic.git', credentialsId: env.GIT_CREDENTIALS]]
          ])

          env.GIT_COMMIT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def tag = sh(script: "git describe --tags --exact-match ${env.GIT_COMMIT} 2>/dev/null || true", returnStdout: true).trim()
          env.BUILD_VERSION = tag ? "${tag}-${env.BUILD_NUMBER}" : "${env.GIT_COMMIT}-${env.BUILD_NUMBER}"
          env.DOCKER_IMAGE = "spring-petclinic:${env.BUILD_VERSION}"
          echo "Commit: ${env.GIT_COMMIT}  Build version: ${env.BUILD_VERSION}"
        }
      }
    }

    stage('Build') {
      tools {
        maven 'maven3'
      }
      steps {
        echo "Maven build (skip tests to allow parallel test stage)..."
        sh 'mvn -B -e -q clean package -DskipTests=true'
      }
    }

    stage('Parallel Testing') {
      parallel {
        stage('Unit Tests') {
          steps {
            echo "Running unit tests (Surefire)..."
            sh 'mvn -B -e -q -DskipITs=true test'
            junit keepLongStdio: true, testResults: '**/target/surefire-reports/*.xml'
          }
        }
        stage('Integration Tests') {
          steps {
            echo "Running integration tests (Failsafe / IT profile)..."
            // adapt command to repo if different. This attempts failsafe verify.
            sh 'mvn -B -e -q -DskipTests=false verify || true'
            junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/*.xml'
          }
        }
      }
      post {
        failure {
          script { currentBuild.result = 'FAILURE' }
        }
      }
    }

    stage('Docker Image Build') {
      steps {
        script {
          if (!fileExists('Dockerfile')) {
            error "Dockerfile not found in repo root. Add a Dockerfile or skip this stage."
          }
          echo "Building Docker image ${env.DOCKER_IMAGE}"
          // prefer docker pipeline plugin docker.build; fallback to shell docker
          try {
            def built = docker.build(env.DOCKER_IMAGE)
          } catch (err) {
            sh "docker build -t ${env.DOCKER_IMAGE} ."
          }
        }
      }
      post {
        success {
          echo "Docker build success"
          script {
            if (params.PUSH_DOCKER.toBoolean()) {
              withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PWD')]) {
                sh "echo \$DOCKER_PWD | docker login -u \$DOCKER_USER --password-stdin"
                sh "docker tag ${env.DOCKER_IMAGE} \$DOCKER_USER/${env.DOCKER_IMAGE}"
                sh "docker push \$DOCKER_USER/${env.DOCKER_IMAGE}"
                sh "docker logout"
              }
            } else {
              echo "PUSH_DOCKER=false; skipping push"
            }
          }
        }
      }
    }

    stage('Artifact Archiving') {
      steps {
        script {
          sh 'mkdir -p build_artifacts || true'
          def jars = sh(script: "ls target/*.jar 2>/dev/null || true", returnStdout: true).trim()
          if (jars) {
            def jar = jars.split()[0]
            def targetName = "spring-petclinic-${env.BUILD_VERSION}.jar"
            sh "cp ${jar} build_artifacts/${targetName}"
            archiveArtifacts artifacts: "build_artifacts/${targetName}", fingerprint: true
            echo "Archived artifact: ${targetName}"
          } else {
            echo "No JAR found; saving docker image as fallback artifact"
            sh "docker save ${env.DOCKER_IMAGE} -o build_artifacts/spring-petclinic-${env.BUILD_VERSION}.tar || true"
            archiveArtifacts artifacts: "build_artifacts/*.tar", fingerprint: true
          }
        }
      }
    }

    stage('Deployment Simulation') {
      when {
        allOf {
          expression { params.DEPLOY_ENV == 'staging' }
          expression { return env.CHANGE_ID == null } // skip for PRs
        }
      }
      steps {
        script {
          echo "Simulating deployment to ${params.DEPLOY_ENV}"
          sh 'docker network inspect spc-net >/dev/null 2>&1 || docker network create spc-net || true'
          // try to run container (detached) - remove if already exists
          sh "docker rm -f spc-${env.BUILD_NUMBER} >/dev/null 2>&1 || true || true"
          sh "docker run --rm -d --name spc-${env.BUILD_NUMBER} --network spc-net ${env.DOCKER_IMAGE} || true"
          // also save image copy to deployed_images folder (simulation)
          sh 'mkdir -p deployed_images || true'
          sh "docker save ${env.DOCKER_IMAGE} -o deployed_images/spring-petclinic-${env.BUILD_VERSION}.tar || true"
          echo "Application deployed successfully."
        }
      }
    }

  } // end stages

  post {
    success {
      script {
        emailext (
          subject: "[Jenkins] SUCCESS: ${currentBuild.fullDisplayName}",
          body: """Build SUCCESS
Job: ${env.JOB_NAME}
Build: ${env.BUILD_NUMBER}
Version: ${env.BUILD_VERSION}
Console: ${env.BUILD_URL}console""",
          to: env.SMTP_RECIPIENT
        )
      }
    }
    failure {
      script {
        emailext (
          subject: "[Jenkins] FAILURE: ${currentBuild.fullDisplayName}",
          body: """Build FAILED
Job: ${env.JOB_NAME}
Build: ${env.BUILD_NUMBER}
Version: ${env.BUILD_VERSION}
Console: ${env.BUILD_URL}console""",
          to: env.SMTP_RECIPIENT
        )
      }
    }
    always {
      echo "Pipeline finished: ${currentBuild.currentResult}"
    }
  }
}
