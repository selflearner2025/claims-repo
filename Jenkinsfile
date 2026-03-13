
pipeline {
  agent any

  environment {
    NEXUS_REPO_URL = 'http://nexus-nexus-repository-manager.default.svc.cluster.local:8081'
    DOMAIN = 'claims'

    GROUP_ID = 'com.demo.camunda'
    ARTIFACT_ID = 'camunda-models'

    // Jenkins credentials ID for Nexus
    NEXUS_CREDS = 'nexus-credentials'
  }
  tools {
  maven 'maven-3.9'
}

  stages {

    stage('Init Variables') {
      steps {
        script {

          // Replace unsafe characters from branch names
          def safeBranch = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-')

          env.VERSION = "${safeBranch}-${env.BUILD_NUMBER}-${new Date().format('yyyyMMdd-HHmmss')}"
          env.ARTIFACT_NAME = "${ARTIFACT_ID}-${env.VERSION}.tar.gz"

          echo "Branch: ${env.BRANCH_NAME}"
          echo "Safe Branch: ${safeBranch}"
          echo "Artifact Name: ${env.ARTIFACT_NAME}"
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          def scmVars = checkout scm

          // Ensure tags exist
          sh 'git fetch --tags --force'

          env.GIT_COMMIT = scmVars.GIT_COMMIT ?: ""
          env.GIT_PREVIOUS_SUCCESSFUL_COMMIT = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: ""

          echo "Current Commit: ${env.GIT_COMMIT}"
          echo "Previous Successful Commit: ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: 'None'}"
        }
      }
    }

    stage('Detect Changed Models') {
      when {
        anyOf {
          branch 'dev'
          branch 'develop'
          branch 'qa'
          branch 'master'
          expression { env.BRANCH_NAME?.startsWith('release/') }
        }
      }

      steps {
        script {

          def baseCommit = null

          if (env.BRANCH_NAME?.startsWith('release/')) {

            def lastProdTag = sh(
              script: "git tag --list 'prod-*' --sort=-version:refname | head -1",
              returnStdout: true
            ).trim()

            if (lastProdTag) {
              echo "Last prod tag found: ${lastProdTag}"
              baseCommit = lastProdTag
            } else {
              echo "No prod tag found. Using previous successful commit."
              baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
            }

          } else {
            baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          }

          if (!baseCommit) {
            echo "First build detected. Using HEAD~1."
            baseCommit = sh(
              script: "git rev-parse HEAD~1",
              returnStdout: true
            ).trim()
          }

          echo "Base commit: ${baseCommit}"
          echo "Head commit: ${env.GIT_COMMIT}"

          def changedFiles = sh(
            script: """
              git diff --name-only --diff-filter=AM ${baseCommit} ${env.GIT_COMMIT} |
              grep -E '\\.(bpmn|dmn)\$' || true
            """,
            returnStdout: true
          ).trim()

          if (!changedFiles) {
            echo "No BPMN/DMN changes detected."
            env.CHANGED_FILES = ""
          } else {
            echo "Changed files:"
            echo changedFiles
            env.CHANGED_FILES = changedFiles
          }
        }
      }
    }

    stage('Package Models') {
      when {
        expression { env.CHANGED_FILES?.trim() }
      }

      steps {
        script {

          sh """
          rm -rf package
          mkdir -p package

          echo "${env.CHANGED_FILES}" | while read file
          do
            mkdir -p package/\$(dirname \$file)
            cp \$file package/\$file
          done

          tar -czf ${ARTIFACT_NAME} -C package .
          """

          sh "ls -lh ${ARTIFACT_NAME}"
        }
      }
    }

    stage('Upload to Nexus') {
      when {
        expression { env.CHANGED_FILES?.trim() }
      }

      steps {

        withCredentials([usernamePassword(
          credentialsId: "${NEXUS_CREDS}",
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {

          sh """
          mvn deploy:deploy-file \
            -DgroupId=${GROUP_ID} \
            -DartifactId=${ARTIFACT_ID} \
            -Dversion=${VERSION} \
            -Dpackaging=tar.gz \
            -Dfile=${ARTIFACT_NAME} \
            -DrepositoryId=nexus \
            -Durl=${NEXUS_REPO_URL} \
            -DgeneratePom=true \
            -Dusername=${NEXUS_USER} \
            -Dpassword=${NEXUS_PASS}
          """
        }
      }
    }

  }

  post {
    success {
      echo "Build completed successfully."
    }
    failure {
      echo "Build failed."
    }
  }
}

