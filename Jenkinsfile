pipeline {
  agent any                    // run on any available agent
    stages {
      stage('checkout') {
        steps { checkout scm }
      }
      stage('build') {
        steps { sh './build.sh' }
      }
      stage('test') {
        steps {
          sh './run-tests.sh'
          junit '**/test-results/*.xml'
        }
      }
    }
}
