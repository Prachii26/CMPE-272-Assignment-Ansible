pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        bat 'echo Hello from Jenkins on Windows'
      }
    }
  }
  post {
    success { echo 'Build succeeded' }
    failure { echo 'Build failed' }
  }
}
