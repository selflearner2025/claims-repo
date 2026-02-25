pipeline {
  agent any

 

  environment {
    NEXUS_REPO_URL = 'http://demo.demo.ins:8081/repository/sameem-bpmn-artifacts'
    DOMAIN = 'claims'
  }

  stages {

    stage('Init Variables') {
      steps {
        script {
          env.ARTIFACT_NAME = "camunda-models-${env.BRANCH_NAME}-${env.BUILD_NUMBER}-${new Date().format('yyyyMMdd-HHmmss')}.tar.gz"
          echo "Artifact Name: ${env.ARTIFACT_NAME}"
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          def scmVars = checkout scm

          // 🔥 IMPORTANT: Ensure tags are available
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

          // -------------------------------------------------
          // RELEASE BRANCH LOGIC
          // -------------------------------------------------
          if (env.BRANCH_NAME?.startsWith('release/')) {

            def lastProdTag = sh(
              script: "git tag --list 'prod-*' --sort=-creatordate | head -1",
              returnStdout: true
            ).trim()

            if (lastProdTag) {
              echo "Last prod tag found: ${lastProdTag}"
              baseCommit = lastProdTag
            } else {
              echo "⚠ No prod-* tags found. Falling back to previous successful commit."
              baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
            }

          } else {
            // -------------------------------------------------
            // NON-RELEASE BRANCH LOGIC
            // -------------------------------------------------
            baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          }

          // -------------------------------------------------
          // HANDLE FIRST BUILD SAFELY
          // -------------------------------------------------
          if (!baseCommit) {
            echo "⚠ No base commit available (likely first build)."
            echo "Using previous commit (HEAD~1) as fallback."

            baseCommit = sh(
              script: "git rev-parse HEAD~1",
              returnStdout: true
            ).trim()
          }

          echo "Base commit: ${baseCommit}"
          echo "Head commit: ${env.GIT_COMMIT}"

          // -------------------------------------------------
          // DETECT CHANGED BPMN/DMN FILES
          // -------------------------------------------------
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
            echo "Changed model files:"
            echo "${changedFiles}"
            env.CHANGED_FILES = changedFiles
          }
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