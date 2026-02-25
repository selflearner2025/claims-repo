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
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          def scmVars = checkout scm

          // Persist required values safely
          env.GIT_COMMIT = scmVars.GIT_COMMIT ?: ""
          env.GIT_PREVIOUS_SUCCESSFUL_COMMIT = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: ""
        }
      }
    }

    stage('Detect Changed Models') {
      when {
        anyOf {
          branch 'dev'          // <-- change if needed
          branch 'develop'
          branch 'qa'
          branch 'master'
          expression { env.BRANCH_NAME?.startsWith('release/') }
        }
      }

      steps {
        script {
          def baseCommit

          if (env.BRANCH_NAME?.startsWith('release/')) {

            def lastProdTag = sh(
              script: "git tag --sort=-creatordate | grep '^prod-' | head -1 || true",
              returnStdout: true
            ).trim()
            echo "Last prod tag: ${lastProdTag}"

            echo "Last prod tag: ${lastProdTag ?: 'none found'}"
            baseCommit = lastProdTag ?: env.GIT_PREVIOUS_SUCCESSFUL_COMMIT

          } else {
            baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          }

          if (!baseCommit) {
            echo "No base commit found — likely first build"
            env.CHANGED_FILES = ""
            return
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

          echo "Changed files: ${changedFiles ?: '(none)'}"
          env.CHANGED_FILES = changedFiles
        }
      }
    }
  }
}