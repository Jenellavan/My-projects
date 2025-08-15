pipeline {
  agent { label 'infra-build-node' }

  options { timestamps() }

  environment {
    // SonarQube (configure the server named "SonarQube" in Manage Jenkins > System)
    SONARQUBE_SERVER = 'SonarQube'
    SONAR_URL        = 'http://54.224.144.227:9000/'

    // Nexus
    NEXUS_URL  = 'http://52.202.80.121:8081'    // no trailing slash
    NEXUS_REPO = 'ezlearn-release'              // must match <server><id> in settings.xml

    // Deploy target (Tomcat host)
    DEPLOY_SERVER  = 'ubuntu@172.31.87.178'     // private IP of Tomcat box
    DEPLOY_PATH    = '/var/lib/tomcat10/webapps'
    TOMCAT_SERVICE = 'tomcat10'

    // Build toolchain
    MAVEN_HOME = '/opt/maven'
    VERSION = '1.0.0'

    // Reports
    SONAR_JACOCO_REPORT_PATH = 'target/site/jacoco/jacoco.xml'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Tooling') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B -v' }
    }

    stage('Build & Unit Tests') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B clean test' }
      post { always { junit 'target/surefire-reports/**/*.xml' } }
    }

    stage('Checkstyle') {
      steps {
        sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B checkstyle:checkstyle'
        archiveArtifacts artifacts: 'target/checkstyle-result.xml', allowEmptyArchive: true
      }
    }

    stage('Coverage') {
      steps {
        sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B jacoco:report'
        archiveArtifacts artifacts: 'target/site/jacoco/**/*', allowEmptyArchive: true
      }
    }

    stage('Preflight: Sonar & SSH') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh '''
            set -e
            echo "Checking Sonar at: ${SONAR_URL}"
            curl -sS --max-time 5 "${SONAR_URL}api/server/version" || { echo "Cannot reach ${SONAR_URL}"; exit 2; }
          '''
        }
        sshagent (credentials: ['ssh-agent-key']) {
          sh '''
            set -e
            echo "Checking SSH to ${DEPLOY_SERVER}..."
            ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no ${DEPLOY_SERVER} 'echo deploy-ssh-ok'
          '''
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh '''
            set -e
            export PATH=$MAVEN_HOME/bin:$PATH
            mvn -B verify sonar:sonar \
              -Dsonar.projectKey=ezlearn \
              -Dsonar.host.url='${SONAR_URL}' \
              ${SONAR_AUTH_TOKEN:+-Dsonar.login="$SONAR_AUTH_TOKEN"} \
              -Dsonar.coverage.jacoco.xmlReportPaths='${SONAR_JACOCO_REPORT_PATH}' \
              -Dsonar.java.binaries=target/classes \
              -Dsonar.sources=src/main/java \
              -Dsonar.ws.timeout=60
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps { timeout(time: 1, unit: 'HOURS') { waitForQualityGate abortPipeline: true } }
    }

    stage('Package WAR') {
      steps {
        sh '''
          set -e
          export PATH=$MAVEN_HOME/bin:$PATH
          mvn -B -DskipTests package
          cp target/ezlearn-${VERSION}.war target/ezlearn.war 2>/dev/null || cp target/*.war target/ezlearn.war
        '''
      }
    }

    stage('Publish to Nexus') {
      steps {
        script {
          def ts = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
          def warName = "ezlearn-${ts}.war"
          sh "cp target/ezlearn.war target/${warName}"

          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh """
              set -e
              export PATH=$MAVEN_HOME/bin:$PATH
              mvn -B deploy:deploy-file \
                -DgroupId=com.ezlearn \
                -DartifactId=ezlearn \
                -Dversion=${ts} \
                -Dpackaging=war \
                -Dfile=target/${warName} \
                -DrepositoryId=${NEXUS_REPO} \
                -Durl=${NEXUS_URL}/repository/${NEXUS_REPO} \
                -DgeneratePom=false \
                --settings jenkins/settings.xml
            """
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        sshagent (credentials: ['ssh-agent-key']) {
          sh """
            set -e
            cp target/ezlearn.war target/ROOT.war
            scp -o StrictHostKeyChecking=no target/ROOT.war ${DEPLOY_SERVER}:/tmp/ROOT.war
            ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} '\
              sudo mv /tmp/ROOT.war ${DEPLOY_PATH}/ROOT.war && \
              sudo chown tomcat:tomcat ${DEPLOY_PATH}/ROOT.war && \
              sudo systemctl restart ${TOMCAT_SERVICE} \
            '
          """
        }
      }
    }
  }

  post {
    always { cleanWs() }
    success { echo "✅ Pipeline executed successfully!" }
    failure { echo "❌ Pipeline failed!" }
  }
}

     
