<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: When conditions and branch logic
pipeline {
  agent any

  stages {
    stage("One") {
      steps {
        echo "Hello"
      }
    }
    stage("Evaluate Master") {
      when {
        branch "master"
      }
      steps {
        echo "World"
        echo "Heal it"
      }
    }
    stage("Branch Test") {
      when {
        not {
          branch "master"
        }
      }
      steps {
        echo "Not master branch"
      }
    }
    stage("Expression Test") {
      when {
        expression {
          echo "Should I run?"
          return false
        }
      }
      steps {
        echo "This should be skipped"
      }
    }
  }
}
    ]]>