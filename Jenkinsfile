pipeline {
  agent any
  options { timestamps() }

  environment {
    DEFAULT_BASE = "origin/main"   // change to origin/master if needed
  }

  stages {
    stage('Checkout') {
      options { skipDefaultCheckout(true) }
      steps {
        checkout scm
        bat 'git --version'
      }
    }

    stage('Detect changes') {
      steps {
        script {
          bat 'git fetch --all --prune'

          def base = env.CHANGE_TARGET ? "origin/${env.CHANGE_TARGET}" : env.DEFAULT_BASE

          // Get changed files (CMD). If nothing changed, output will be empty.
          def diffOut = bat(
            script: "@echo off\r\ngit diff --name-only ${base}...HEAD",
            returnStdout: true
          ).trim()

          // Jenkins on Windows may include extra CR; normalize
          diffOut = diffOut.replace("\r", "")

          def files = diffOut ? diffOut.split("\n") as List : []

          echo "Base for diff: ${base}"
          echo "Changed files:\n- " + (files ? files.join("\n- ") : "(none)")

          env.RUN_MODULE1 = files.any { it.startsWith("module1/") }.toString()
          env.RUN_MODULE2 = files.any { it.startsWith("module2/") }.toString()

          echo "RUN_MODULE1=${env.RUN_MODULE1}"
          echo "RUN_MODULE2=${env.RUN_MODULE2}"
        }
      }
    }

    stage('Run module pipelines') {
      steps {
        script {
          def branches = [:]

          if (env.RUN_MODULE1 == "true") {
            branches["module1"] = {
              def m1 = load("module1/Jenkinsfile_module1")
              m1.run()
            }
          }

          if (env.RUN_MODULE2 == "true") {
            branches["module2"] = {
              def m2 = load("module2/Jenkinsfile_module2")
              m2.run()
            }
          }

          if (branches.isEmpty()) {
            echo "No changes in module1/ or module2/. Nothing to run."
          } else {
            parallel branches
          }
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished."
    }
  }
}
