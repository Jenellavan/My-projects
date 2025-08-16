pipeline {
    agent { label 'infra-build-node' }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://3.93.15.96:8081'
        NEXUS_REPO = 'ezlearn-release'
        CONTAINER_NAME = 'ezlearn'
        APP_PORT = '8081'          // Must match the port exposed in your Dockerfile
        VERSION = '1.0.0'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { sh 'mvn clean compile' }
        }

        stage('Unit Test') {
            steps { sh 'mvn test' }
        }

        stage('Checkstyle Analysis') {
            steps { sh 'mvn checkstyle:check' }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                      mvn sonar:sonar \
                        -Dsonar.projectKey=ezlearn \
                        -Dsonar.host.url=http://54.163.53.76:9000
                    '''
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                  mvn -DskipTests package
                  # ensure predictable name for Dockerfile COPY
                  cp target/ezlearn-*.war target/ezlearn.war
                '''
            }
        }

        // Optional: keep publishing the WAR to Nexus for artifact history
        stage('Publish to Nexus') {
            steps {
                script {
                    def ts = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    def warName = "ezlearn-${ts}.war"
                    def warPath = "target/${warName}"
                    sh "cp target/ezlearn.war ${warPath}"

                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        def mvnCmd = "mvn deploy:deploy-file" +
                                     " -DgroupId=com.ezlearn" +
                                     " -DartifactId=ezlearn" +
                                     " -Dversion=${ts}" +
                                     " -Dpackaging=war" +
                                     " -Dfile=${warPath}" +
                                     " -DrepositoryId=ezlearn-release" +
                                     " -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/" +
                                     " -DgeneratePom=true" +
                                     " --settings jenkins/settings.xml"
                        sh mvnCmd
                    }
                }
            }
        }

        stage('Build Docker Image (Tomcat + WAR)') {
            steps {
                script {
                    env.IMAGE_TAG = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    env.IMAGE_NAME = "ezlearn:${IMAGE_TAG}"

                    sh """
                      docker build \
                        --build-arg WAR_FILE=target/ezlearn.war \
                        -t ${IMAGE_NAME} .
                    """
                }
            }
        }

        stage('Run Container (On Slave)') {
            steps {
                script {
                    sh """
                      docker rm -f ${CONTAINER_NAME} || true
                      docker run -d --name ${CONTAINER_NAME} \
                        -p ${APP_PORT}:${APP_PORT} \
                        --restart=always \
                        ${IMAGE_NAME}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                def hasReports = fileExists('target/surefire-reports')
                if (hasReports) {
                    junit 'target/surefire-reports/*.xml'
                } else {
                    echo "No test reports found to archive."
                }
            }
        }
        success { echo "✅ Pipeline executed successfully! (Container running on ${env.NODE_NAME}:${APP_PORT})" }
        failure { echo "❌ Pipeline failed!" }
    }
}
