pipeline {
  agent any

  tools {
    jdk 'jdk-17'          // name from Global Tool Config
    maven 'Maven-3'
  }

  environment {
    TOMCAT_HOST = '100.28.131.81'
    TOMCAT_USER = 'ubuntu'
    WEBAPPS_DIR = '/opt/tomcat/tomcat10/webapps/'
    SSH_CREDENTIALS_ID = 'ec2-ssh'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']],
          userRemoteConfigs: [[url: 'https://github.com/Harshitha-Galla5/Java-Login-App.git']]])
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B clean package -DskipTests'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
          sh """
            scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_HOST}:${WEBAPPS_DIR}
            ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} 'sudo systemctl restart tomcat'
          """
        }
      }
    }
  }

  post {
    success {
      echo 'Deployed successfully'
    }
    failure {
      echo 'Deployment failed'
    }
  }
}
