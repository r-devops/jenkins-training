pipeline {
  agent any                    // run on any available agent
  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
    choice(name: 'ENV',   choices: ['dev','staging','prod'], description: 'Target env')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip the test stage')
  }
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
