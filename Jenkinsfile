// OPTIMIZED: shared gem cache, parallel checks, auto-cancel of old builds,
// timeout, log retention, and the Docker build only on main.
pipeline {
  agent none

  options {
    timestamps()
    timeout(time: 20, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds(abortPrevious: true)
  }

  triggers { pollSCM('H/2 * * * *') }

  stages {
    stage('Install gems (cached)') {
      agent { docker { image 'ruby:3.3'; args '-v gem-cache:/usr/local/bundle' } }
      steps { retry(5) { sh 'bundle install --jobs 4 --quiet --retry 5' } }
    }

    stage('Checks') {
      parallel {
        stage('Lint') {
          agent { docker { image 'ruby:3.3'; args '-v gem-cache:/usr/local/bundle' } }
          steps {
            sh 'bundle install --local --quiet'
            sh 'bundle exec rubocop'
          }
        }
        stage('Security') {
          agent { docker { image 'ruby:3.3'; args '-v gem-cache:/usr/local/bundle' } }
          steps {
            sh 'bundle install --local --quiet'
            sh 'bundle exec brakeman -q --no-pager'
            sh 'gem install bundler-audit && bundle-audit check --update'
          }
        }
        stage('Test') {
          agent { docker { image 'ruby:3.3'; args '-v gem-cache:/usr/local/bundle' } }
          environment { RAILS_ENV = 'test' }
          steps {
            sh 'bundle install --local --quiet'
            sh 'bin/rails db:test:prepare test'
          }
        }
      }
    }

    stage('Docker build') {
      when { expression { env.GIT_BRANCH == 'origin/main' } }
      agent any
      steps { sh 'docker build -t rails-app:${BUILD_NUMBER} .' }
    }
  }
}
