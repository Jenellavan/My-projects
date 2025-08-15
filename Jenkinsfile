pipeline {
  agent { label 'infra-build-node' }

  environment {
    SONARQUBE_SERVER = 'SonarQube'                    // Name from Manage Jenkins > System > SonarQube servers
    SONAR_HOST_URL_EXPECTED = 'http://54.224.144.227:9000'
    NEXUS_URL = 'http://54.172.175.151:8081'         // no trailing slash to avoid //
    NEXUS_REPO = 'ezlearn-release'                   // repo name in Nexus (path piece)
    DEPLOY_SERVER = 'ubuntu@18.207.173.147'
    DEPLOY_PATH = '/opt/tomcat/webapps'
    VERSION = '1.0.0'
    MAVEN_HOME = '/opt/maven'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Check Maven Version') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B -v' }
    }

    stage('Build') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B clean compile' }
    }

    stage('Unit Test') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B test' }
      post { always { junit 'target/surefire-reports/*.xml' } } // collect right after tests
    }

    stage('Checkstyle Analysis') {
      steps { sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -B checkstyle:check' }
    }

    stage('SonarQube: Preflight') {
      steps {
        // Make sure we’re actually pointing at the right Sonar URL and it’s reachable
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh '''
            echo "Injected SONAR_HOST_URL=$SONAR_HOST_URL"
            if [ "$SONAR_HOST_URL" != "${SONAR_HOST_URL_EXPECTED}" ]; then
              echo "ERROR: SONAR_HOST_URL is $SONAR_HOST_URL but expected ${SONAR_HOST_URL_EXPECTED}" >&2
              exit 2
            fi
            curl -sS --max-time 5 "$SONAR_HOST_URL/api/server/version" || {
              echo "ERROR: Cannot reach $SONAR_HOST_URL (version endpoint failed)"; exit 3; }
          '''
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // Let the plugin inject URL + token; don’t hardcode -Dsonar.host.url
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh '''
            export PATH=$MAVEN_HOME/bin:$PATH
            mvn -B verify sonar:sonar \
              -Dsonar.projectKey=ezlearn \
              -Dsonar.ws.timeout=60
          '''
        }
      }
    }

    stage('Package WAR') {
      steps {
        sh '''
          export PATH=$MAVEN_HOME/bin:$PATH
          mvn -B -DskipTests package
          cp target/ezlearn-${VERSION}.war target/ezlearn.war || cp target/*.war target/ezlearn.war
        '''
      }
    }

    stage('Publish to Nexus') {
      steps {
        script {
          def ts = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
          def warName = "ezlearn-${ts}.war"
          sh "cp target/ezlearn.war target/${warName}"

          // IMPORTANT: repositoryId must match a <server><id>...</id></server> in jenkins/settings.xml
          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh """
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
            cp target/ezlearn.war target/ROOT.war
            scp -o StrictHostKeyChecking=no target/ROOT.war ${DEPLOY_SERVER}:/tmp/ROOT.war
            ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} 'sudo mv /tmp/ROOT.war ${DEPLOY_PATH}/ROOT.war && sudo chown tomcat:tomcat ${DEPLOY_PATH}/ROOT.war'
          """
        }
      }
    }
  }

  post {
    success { echo "✅ Pipeline executed successfully!" }
    failure { echo "❌ Pipeline failed!" }
  }
}
