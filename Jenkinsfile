pipeline {
  agent any

  tools {
    maven 'maven-3.9'
  }

  environment {
    NEXUS_REPO_URL = 'http://nexus-nexus-repository-manager.default.svc.cluster.local:8081/repository/sample-repo'
    GROUP_ID = 'com.demo.camunda'
    ARTIFACT_ID = 'camunda-models'
    DOMAIN = 'claims'
    NEXUS_CREDS = 'nexus-credentials'
  }

  stages {

    stage('Init Variables') {
      steps {
        script {
          def safeBranch = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-')
          def timestamp = new Date().format('yyyyMMdd-HHmmss')

          env.VERSION = "${safeBranch}-${env.BUILD_NUMBER}-${timestamp}"
          env.ARTIFACT_NAME = "${env.ARTIFACT_ID}-${env.VERSION}.tar.gz"

          echo "Branch: ${env.BRANCH_NAME}"
          echo "Version: ${env.VERSION}"
          echo "Artifact: ${env.ARTIFACT_NAME}"
        }
      }
    }

    stage('Checkout') {
      steps {
        script {
          def scmVars = checkout scm

          sh '''
            git fetch --tags --force
            git fetch --unshallow || true
          '''

          env.GIT_COMMIT = scmVars.GIT_COMMIT ?: sh(
            script: "git rev-parse HEAD",
            returnStdout: true
          ).trim()

          env.GIT_PREVIOUS_SUCCESSFUL_COMMIT =
            scmVars.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: ""

          echo "Current Commit: ${env.GIT_COMMIT}"
          echo "Previous Successful Commit: ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: 'None'}"
        }
      }
    }

    stage('Detect Changed Models') {
      when {
        expression {
          env.BRANCH_NAME in ['dev','develop','qa','master'] ||
          env.BRANCH_NAME.startsWith('release/')
        }
      }

      steps {
        script {

          def baseCommit = ""

          if (env.BRANCH_NAME.startsWith('release/')) {

            def lastProdTag = sh(
              script: "git tag --list 'prod-*' --sort=-v:refname | head -1",
              returnStdout: true
            ).trim()

            if (lastProdTag) {
              echo "Using last prod tag: ${lastProdTag}"
              baseCommit = lastProdTag
            }

          } else {
            baseCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
          }

          if (!baseCommit) {
            echo "First build detected. Using HEAD~1"
            baseCommit = sh(
              script: "git rev-parse HEAD~1",
              returnStdout: true
            ).trim()
          }

          echo "Base Commit: ${baseCommit}"
          echo "Head Commit: ${env.GIT_COMMIT}"

          def changedFiles = sh(
            script: """
              git diff --name-only --diff-filter=AM ${baseCommit} ${env.GIT_COMMIT} \
              | grep -E '\\.(bpmn|dmn)\$' || true
            """,
            returnStdout: true
          ).trim()

          if (!changedFiles) {
            echo "No BPMN/DMN changes detected."
            env.CHANGED_FILES = ""
          } else {
            echo "Changed Models:"
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

          printf "%s\\n" "${env.CHANGED_FILES}" | while IFS= read -r file
          do
            mkdir -p package/\$(dirname "\$file")
            cp "\$file" package/"\$file"
          done

          tar -czf ${env.ARTIFACT_NAME} -C package .
          """

          sh "ls -lh ${env.ARTIFACT_NAME}"

          archiveArtifacts artifacts: "${env.ARTIFACT_NAME}", fingerprint: true
        }
      }
    }

    stage('Upload to Nexus') {
      when {
        expression { env.CHANGED_FILES?.trim() }
      }

      steps {

        withCredentials([
          usernamePassword(
            credentialsId: "${env.NEXUS_CREDS}",
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )
        ]) {

          sh """
          echo $NEXUS_USER:$NEXUS_PASS 
          mvn deploy:deploy-file \
            -DgroupId=${env.GROUP_ID} \
            -DartifactId=${env.ARTIFACT_ID} \
            -Dversion=${env.VERSION} \
            -Dpackaging=tar.gz \
            -Dfile=${env.ARTIFACT_NAME} \
            -DrepositoryId=sample-repo \
            -Durl=${env.NEXUS_REPO_URL} \
            -DgeneratePom=true \
            -Dusername=\$NEXUS_USER \
            -Dpassword=\$NEXUS_PASS
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