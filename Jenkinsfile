// CI/CD pipeline for Dr. Shawn's pet clinic (Spring PetClinic) - Bella Solutions
// Git (GitHub) -> Maven build + tests on JDK 11 -> WAR -> local Tomcat 9 on the same EC2 instance
pipeline {
    agent any

    tools {
        jdk 'jdk11'      // Manage Jenkins > Tools: /usr/lib/jvm/java-11-amazon-corretto
        maven 'maven3'   // Manage Jenkins > Tools: /opt/maven
    }

    environment {
        TOMCAT_WEBAPPS = '/opt/tomcat/webapps'
        APP_URL        = 'http://localhost:8090/petclinic/'
    }

    triggers {
        // Check GitHub for new commits every ~2 minutes (Jenkins is not reachable by GitHub webhooks)
        pollSCM('H/2 * * * *')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --format="Commit %h by %an: %s"'
            }
        }

        stage('Build') {
            steps {
                sh 'java -version && mvn -version'
                sh 'mvn -B clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B -DskipTests package'
                archiveArtifacts artifacts: 'target/petclinic.war', fingerprint: true
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                // Tomcat hot-deploys any WAR copied into webapps/; copy then rename so it never sees a half-written file
                sh '''
                    cp target/petclinic.war "$TOMCAT_WEBAPPS/.petclinic.war.tmp"
                    mv -f "$TOMCAT_WEBAPPS/.petclinic.war.tmp" "$TOMCAT_WEBAPPS/petclinic.war"
                '''
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        code=$(curl -s -o /dev/null -w '%{http_code}' "$APP_URL" || true)
                        [ "$code" = 200 ] && echo "PetClinic is live: $APP_URL (HTTP $code)" && exit 0
                        sleep 5
                    done
                    echo "PetClinic did not come up"; exit 1
                '''
            }
        }
    }

    post {
        success { echo 'Deployed: open port 8090, path /petclinic/, on the EC2 public IP' }
        failure { echo 'Pipeline failed - the previous version stays deployed in Tomcat.' }
    }
}
