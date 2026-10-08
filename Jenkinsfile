// ============================================================
// Pipeline 3 : Maven build -> SonarQube code quality -> Nexus artifact upload
// Flow       : Checkout -> Compile -> Unit Test -> SonarQube -> Quality Gate -> Nexus
// Rule       : Compile / Test / Quality Gate fail ayite Nexus ki emi upload avvadu.
// ============================================================
pipeline {
    agent any

    tools {
        maven 'maven3'   // Manage Jenkins > Tools lo ichina Maven name
        jdk   'jdk17'    // Manage Jenkins > Tools lo ichina JDK name
    }

    environment {
        // ---- REPLACE: mee project values ----
        REPO              = 'github.com/YOUR-USER/YOUR-MAVEN-REPO.git'
        BRANCH            = 'main'
        SONAR_PROJECT_KEY = 'my-maven-app'

        // ---- REPLACE: mee Nexus URL (last lo slash lekunda) ----
        NEXUS_URL         = 'http://NEXUS-HOST:8081'
        NEXUS_RELEASES    = 'maven-releases'
        NEXUS_SNAPSHOTS   = 'maven-snapshots'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                git branch: "${env.BRANCH}", url: "https://${env.REPO}", credentialsId: 'github-creds'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                // jacoco plugin pom.xml lo unte coverage SonarQube ki velthundi
                sh 'mvn -B test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // 'sonarqube' = Manage Jenkins > System > SonarQube servers lo ichina name
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn -B verify sonar:sonar \
                            -DskipTests \
                            -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                            -Dsonar.host.url="$SONAR_HOST_URL" \
                            -Dsonar.token="$SONAR_AUTH_TOKEN"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                // SonarQube lo Jenkins webhook undali: <JENKINS_URL>/sonarqube-webhook/
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                                 usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        set -e

                        # Maven settings: password file lo raayakunda env nunchi teesukuntundi
                        cat > settings-nexus.xml <<'XML'
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
XML

                        # SNAPSHOT version aithe snapshots repo, lekapothe releases repo
                        VERSION=$(mvn -B -q help:evaluate -Dexpression=project.version -DforceStdout)
                        case "$VERSION" in
                            *-SNAPSHOT) TARGET_REPO="$NEXUS_SNAPSHOTS" ;;
                            *)          TARGET_REPO="$NEXUS_RELEASES" ;;
                        esac
                        echo "Version: $VERSION  ->  Nexus repo: $TARGET_REPO"

                        mvn -B -s settings-nexus.xml -DskipTests \
                            clean package \
                            org.apache.maven.plugins:maven-deploy-plugin:3.1.2:deploy \
                            -DaltDeploymentRepository="nexus::${NEXUS_URL}/repository/${TARGET_REPO}/"

                        rm -f settings-nexus.xml
                    '''
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '**/target/*.jar, **/target/*.war', allowEmptyArchive: true, fingerprint: true
            echo "SUCCESS: Compile, SonarQube Quality Gate pass ayyayi. Artifact Nexus lo save ayyindi: ${env.NEXUS_URL}"
        }
        failure {
            echo 'FAILED: Console Output chudandi (Compile / Test / Sonar / Nexus upload errors).'
        }
        aborted {
            echo 'ABORTED: Quality Gate fail ayyindi leda build cancel chesaru. Nexus ki emi upload avvaledu.'
        }
        always {
            sh 'rm -f settings-nexus.xml || true'
        }
    }
}
