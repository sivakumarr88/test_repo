pipeline {
  agent any
  options { timestamps() }

  environment {
    DEFAULT_BASE = "origin/main"   // change to origin/master if your default branch is master
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git --version'
      }
    }

    stage('Detect changes') {
      steps {
        script {
          sh 'git fetch --all --prune'

          // If this is a PR build and CHANGE_TARGET is available, diff against that branch.
          def base = env.CHANGE_TARGET ? "origin/${env.CHANGE_TARGET}" : env.DEFAULT_BASE

          // List changed files between base and current HEAD, with a fallback for single-commit cases
          def diffOut = isUnix()
            ? sh(script: "git diff --name-only ${base}...HEAD", returnStdout: true).trim()
            : bat(script: "@echo off\r\ngit diff --name-only ${base}...HEAD", returnStdout: true).trim()

          if (!diffOut) {
            diffOut = isUnix()
              ? sh(script: """
                  git rev-parse --verify HEAD~1 >/dev/null 2>&1
                  if [ $? -eq 0 ]; then
                    git diff --name-only HEAD~1..HEAD
                  else
                    git show --pretty='' --name-only HEAD
                  fi
                """, returnStdout: true).trim()
              : bat(script: """@echo off
                  git rev-parse --verify HEAD~1 >nul 2>nul
                  if %ERRORLEVEL% EQU 0 (
                    git diff --name-only HEAD~1..HEAD
                  ) else (
                    git show --pretty="" --name-only HEAD
                  )
                """, returnStdout: true).trim()
          }

          def files = diffOut ? diffOut.split("\n") as List : []

          echo "Base for diff: ${base}"
          echo "Changed files:\n- " + (files ? files.join("\n- ") : "(none)")

          // Flags for modules
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
