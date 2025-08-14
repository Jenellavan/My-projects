pipeline {
    agent { label 'infra-build-node' }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://nexus.mitechnology.org:8081'
        NEXUS_REPO = 'ezlearn-release'
        DEPLOY_SERVER = 'ubuntu@52.207.238.92'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        VERSION = '1.0.0'
        SONAR_JACOCO_REPORT_PATH = 'target/site/jacoco/jacoco.xml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Static Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
                archiveArtifacts artifacts: 'target/checkstyle-result.xml', allowEmptyArchive: true
            }
        }

        stage('Unit Test & Coverage') {
            steps {
                sh 'mvn test jacoco:report'
                junit 'target/surefire-reports/**/*.xml'
                archiveArtifacts artifacts: 'target/site/jacoco/**/*.xml', allowEmptyArchive: true
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=ezlearn \
                          -Dsonar.host.url=http://54.224.144.227:9000 \
                          -Dsonar.qualitygate.wait=true \
                          -Dsonar.coverage.jacoco.xmlReportPaths=${SONAR_JACOCO_REPORT_PATH} \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.sources=src/main/java
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                    mvn package
                    cp target/ezlearn-${VERSION}.war target/ezlearn.war
                '''
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    def warName = "ezlearn-${timestamp}.war"
                    def warPath = "target/${warName}"

                    sh "cp target/ezlearn.war ${warPath}"

                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                            mvn deploy:deploy-file \
                              -DgroupId=com.ezlearn \
                              -DartifactId=ezlearn \
                              -Dversion=${timestamp} \
                              -Dpackaging=war \
                              -Dfile=${warPath} \
                              -DrepositoryId=ezlearn-release \
                              -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                              -DgeneratePom=true \
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
        always {
            cleanWs()
            script {
                if (currentBuild.result == 'UNSTABLE') {
                    echo "⚠️ Build unstable due to quality gate warnings."
                } else if (currentBuild.result == 'FAILURE') {
                    echo "❌ Build failed!"
                } else {
                    echo "✅ Pipeline executed successfully!"
                }
            }
        }
    }
}
