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
        steps { sh 'echo ./build.sh' }
      }
      stage('test') {
        steps {
          sh 'echo ./run-tests.sh'
          sh 'env'
        }
      }
      stage('deploy prod') {
        when {
          allOf {
            expression { params.BRANCH == 'main' }
            expression { params.ENV == 'prod' }
          }
        }
        steps { sh './deploy-prod.sh' }
      }
    }
}
