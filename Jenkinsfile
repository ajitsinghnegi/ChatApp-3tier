pipeline {
  agent any

  environment {
    SONAR_HOME = tool "sonarqube"
  }

  stages {
    stage('Clean UP') {
      steps {
        cleanWs()
      }
    }
    stage('Code Check out') {
      steps {
        git branch: 'master',
          url: 'https://github.com/iemafzalhassan/full-stack_chatApp.git'
      }
    }
    stage('Code Quality') {
      steps {
        withSonarQubeEnv("sonarqube") {
          sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=chat-app -Dsonar.projectKey=chat-app -X"
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          sh 'docker compose up -d --build'
        }
      }
    }
  }
}
