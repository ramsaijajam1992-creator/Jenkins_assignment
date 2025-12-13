pipeline {
  agent { label 'maven-agent' }

  environment {
    // Update these 3 values for your setup
    APP_SERVER_IP   = "172.31.64.121"
    APP_SERVER_USER = "ubuntu"            // or ec2-user
    SSH_CRED_ID     = "gitcred01"    // Jenkins SSH credential ID for App Server
  }

  stages {

    // 1) Checkout Code
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    // 2) Build & Test using Maven (mvn clean test)
    stage('Build & Test using Maven') {
      steps {
        sh 'mvn clean test'
      }
    }

    // 3) Security Scan Stage (Trivy) + Fail if threshold exceeded
    stage('Security Scan Stage') {
      steps {
        // Threshold: fails build if HIGH or CRITICAL vulnerabilities are found
        sh 'trivy fs . --severity HIGH,CRITICAL --exit-code 1'
      }
    }

    // 4) Package Stage (mvn package) + Archive artifact
    stage('Package Stage') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.jar,target/*.war', fingerprint: true
        }
      }
    }

    // 5) Deploy to Application Server (only main branch)
    stage('Deploy to Application Server') {
      when { branch 'main' }
      steps {
        sshagent([env.SSH_CRED_ID]) {
          sh '''
            set -e

            ARTIFACT=$(ls -1 target/*.jar target/*.war 2>/dev/null | head -n 1)
            if [ -z "$ARTIFACT" ]; then
              echo "No artifact found under target/. Build did not produce jar/war."
              exit 1
            fi

            # Copy artifact to Application Server
            scp -o StrictHostKeyChecking=no "$ARTIFACT" \
              ${APP_SERVER_USER}@${APP_SERVER_IP}:/home/${APP_SERVER_USER}/app.jar

            # Restart application on Application Server
            ssh -o StrictHostKeyChecking=no ${APP_SERVER_USER}@${APP_SERVER_IP} "
              pkill -f app.jar || true
              nohup java -jar /home/${APP_SERVER_USER}/app.jar > app.log 2>&1 &
            "
          '''
        }
      }
    }
  }
}
