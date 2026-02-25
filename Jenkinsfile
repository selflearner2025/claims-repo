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
          scmVars = checkout scm
        }
      }
    }

    stage('Detect Changed Models') {
      when {
        anyOf {
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

            echo "Last prod tag: ${lastProdTag ?: 'none found'}"
            baseCommit = lastProdTag ?: scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT

          } else {

            baseCommit = scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          }

          if (!baseCommit) {
            echo "No base commit found — skipping diff"
            env.CHANGED_FILES = ""
            return
          }

          echo "Base commit: ${baseCommit}"
          echo "Head commit: ${scmVars.GIT_COMMIT}"

          def changedFiles = sh(
            script: """
              git diff --name-only --diff-filter=AM ${baseCommit} ${scmVars.GIT_COMMIT} |
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