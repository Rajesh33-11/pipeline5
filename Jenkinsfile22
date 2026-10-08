```groovy
// ============================================================
// Pipeline : node/ directory JSON files update + Maven Build
//
// Files:
//   dev.json, prod.json, stage.json, uat.json
//
// Rules:
//   1. At least one JSON file must be selected.
//   2. All input fields must have values.
//   3. Selected JSON files will be updated.
//   4. Changes will be committed and pushed automatically.
//   5. Maven build will run after Git push.
//   6. JAR/WAR artifacts will be archived.
// ============================================================

// ------------------------------------------------------------
// Helper: Stop build as ABORTED
// ------------------------------------------------------------
def abortBuild(String msg) {
    currentBuild.result = 'ABORTED'
    error(msg)
}

pipeline {

    agent any

    // --------------------------------------------------------
    // Jenkins Tools
    //
    // These names MUST exactly match:
    // Manage Jenkins → Tools
    //
    // JDK:
    //     JDK17
    //
    // Maven:
    //     Maven3
    // --------------------------------------------------------
    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    // --------------------------------------------------------
    // Parameters
    // --------------------------------------------------------
    parameters {

        // JSON files
        booleanParam(
            name: 'DEV_JSON',
            defaultValue: false,
            description: 'Update node/dev.json'
        )

        booleanParam(
            name: 'PROD_JSON',
            defaultValue: false,
            description: 'Update node/prod.json'
        )

        booleanParam(
            name: 'STAGE_JSON',
            defaultValue: false,
            description: 'Update node/stage.json'
        )

        booleanParam(
            name: 'UAT_JSON',
            defaultValue: false,
            description: 'Update node/uat.json'
        )

        // Values
        string(
            name: 'P_ENVIRONMENT',
            defaultValue: '',
            description: 'environment (e.g. dev)'
        )

        string(
            name: 'P_NODE_NAME',
            defaultValue: '',
            description: 'nodeName (e.g. node-02)'
        )

        string(
            name: 'P_NODE_TYPE',
            defaultValue: '',
            description: 'nodeType (e.g. worker)'
        )

        string(
            name: 'P_REGION',
            defaultValue: '',
            description: 'region (e.g. us-east-1)'
        )

        string(
            name: 'P_AZ',
            defaultValue: '',
            description: 'availabilityZone (e.g. us-east-1b)'
        )

        string(
            name: 'P_INSTANCE_TYPE',
            defaultValue: '',
            description: 'instanceType (e.g. t3.large)'
        )

        string(
            name: 'P_OS',
            defaultValue: '',
            description: 'os (e.g. Amazon Linux 2023)'
        )

        string(
            name: 'P_K8S_ROLE',
            defaultValue: '',
            description: 'kubernetes.role (e.g. worker)'
        )

        string(
            name: 'P_K8S_VERSION',
            defaultValue: '',
            description: 'kubernetes.version (e.g. 1.36)'
        )

        string(
            name: 'P_CPU',
            defaultValue: '',
            description: 'resources.cpu (e.g. 2)'
        )

        string(
            name: 'P_MEMORY',
            defaultValue: '',
            description: 'resources.memory (e.g. 4Gi)'
        )

        string(
            name: 'P_DISK',
            defaultValue: '',
            description: 'resources.disk (e.g. 50Gi)'
        )

        string(
            name: 'P_LABEL_ENV',
            defaultValue: '',
            description: 'labels.environment (e.g. dev)'
        )

        string(
            name: 'P_LABEL_TEAM',
            defaultValue: '',
            description: 'labels.team (e.g. platform)'
        )
    }

    // --------------------------------------------------------
    // Environment
    // --------------------------------------------------------
    environment {

        REPO = 'github.com/Rajesh33-11/flipkart.git'

        BRANCH = 'main'
    }

    // ========================================================
    // STAGES
    // ========================================================
    stages {

        // ----------------------------------------------------
        // 1. Validate Inputs
        // ----------------------------------------------------
        stage('Validate Inputs') {

            steps {

                script {

                    // ----------------------------------------
                    // Check selected JSON files
                    // ----------------------------------------
                    def files = []

                    if (params.DEV_JSON) {
                        files << 'node/dev.json'
                    }

                    if (params.PROD_JSON) {
                        files << 'node/prod.json'
                    }

                    if (params.STAGE_JSON) {
                        files << 'node/stage.json'
                    }

                    if (params.UAT_JSON) {
                        files << 'node/uat.json'
                    }

                    if (files.isEmpty()) {

                        abortBuild(
                            'ABORTED: At least one JSON file must be selected.'
                        )
                    }

                    env.FILES = files.join(' ')

                    // ----------------------------------------
                    // Validate all input values
                    // ----------------------------------------
                    def valueNames = [
                        'P_ENVIRONMENT',
                        'P_NODE_NAME',
                        'P_NODE_TYPE',
                        'P_REGION',
                        'P_AZ',
                        'P_INSTANCE_TYPE',
                        'P_OS',
                        'P_K8S_ROLE',
                        'P_K8S_VERSION',
                        'P_CPU',
                        'P_MEMORY',
                        'P_DISK',
                        'P_LABEL_ENV',
                        'P_LABEL_TEAM'
                    ]

                    def entered = []
                    def missing = []

                    valueNames.each { name ->

                        def value = params[name]

                        if (
                            value == null ||
                            value.toString().trim() == '' ||
                            value.toString().trim() == 'no-change'
                        ) {

                            missing << name

                        } else {

                            entered << "${name}=${value}"
                        }
                    }

                    // ----------------------------------------
                    // Stop if any value is missing
                    // ----------------------------------------
                    if (!missing.isEmpty()) {

                        abortBuild(
                            "ABORTED: Missing values: ${missing.join(', ')}. " +
                            "Please provide values for all fields."
                        )
                    }

                    // ----------------------------------------
                    // CPU validation
                    // CPU must contain only numbers
                    // ----------------------------------------
                    def cpu = params.P_CPU.toString().trim()

                    if (!cpu.matches('[0-9]+')) {

                        abortBuild(
                            "ABORTED: P_CPU must contain only numbers. " +
                            "Received: '${cpu}'"
                        )
                    }

                    // ----------------------------------------
                    // Display information in Jenkins
                    // ----------------------------------------
                    currentBuild.displayName =
                        "#${env.BUILD_NUMBER} ${files.join(', ')}"

                    currentBuild.description =
                        "Files: ${files.join(', ')} | " +
                        "Changes: ${entered.join(', ')}"

                    echo "========================================"
                    echo "Selected Files : ${env.FILES}"
                    echo "Changes        : ${entered.join(', ')}"
                    echo "========================================"
                }
            }
        }

        // ----------------------------------------------------
        // 2. Checkout
        // ----------------------------------------------------
        stage('Checkout') {

            steps {

                cleanWs()

                git(
                    branch: "${env.BRANCH}",
                    url: "https://${env.REPO}",
                    credentialsId: 'github-creds'
                )
            }
        }

        // ----------------------------------------------------
        // 3. Update JSON
        // ----------------------------------------------------
        stage('Update JSON') {

            steps {

                sh '''
                    set -e

                    echo "Creating jq filter..."

                    cat > filter.jq <<'JQ'

def p($k):
    ($ENV[$k] // "");

def setstr($path; $k):
    if p($k) != "" then
        setpath($path; p($k))
    else
        .
    end;

setstr(["environment"]; "P_ENVIRONMENT")
| setstr(["nodeName"]; "P_NODE_NAME")
| setstr(["nodeType"]; "P_NODE_TYPE")
| setstr(["region"]; "P_REGION")
| setstr(["availabilityZone"]; "P_AZ")
| setstr(["instanceType"]; "P_INSTANCE_TYPE")
| setstr(["os"]; "P_OS")
| setstr(["kubernetes","role"]; "P_K8S_ROLE")
| setstr(["kubernetes","version"]; "P_K8S_VERSION")
| setstr(["resources","cpu"]; "P_CPU")
| setstr(["resources","memory"]; "P_MEMORY")
| setstr(["resources","disk"]; "P_DISK")
| setstr(["labels","environment"]; "P_LABEL_ENV")
| setstr(["labels","team"]; "P_LABEL_TEAM")

JQ

                    echo "Updating selected JSON files..."

                    for f in $FILES
                    do

                        if [ ! -f "$f" ]; then

                            echo "ERROR: File not found: $f"

                            exit 1
                        fi

                        jq -f filter.jq "$f" > tmp.json

                        mv tmp.json "$f"

                        echo "========================================"
                        echo "Updated: $f"
                        echo "========================================"

                        cat "$f"

                    done

                    rm -f filter.jq
                '''
            }
        }

        // ----------------------------------------------------
        // 4. Review Changes
        // ----------------------------------------------------
        stage('Review Changes') {

            steps {

                script {

                    def rc = sh(
                        returnStatus: true,
                        script: 'git diff --quiet'
                    )

                    if (rc != 0) {
                        env.HAS_CHANGES = 'true'
                    } else {
                        env.HAS_CHANGES = 'false'
                    }
                }

                sh '''
                    echo "========================================"
                    echo "GIT DIFF"
                    echo "========================================"

                    git --no-pager diff --stat

                    echo "========================================"
                    echo "DETAILED DIFF"
                    echo "========================================"

                    git --no-pager diff
                '''

                script {

                    if (env.HAS_CHANGES == 'false') {

                        echo 'No changes detected.'
                        echo 'Entered values already exist in JSON files.'
                    }
                }
            }
        }

        // ----------------------------------------------------
        // 5. Commit & Push
        // ----------------------------------------------------
        stage('Commit & Push') {

            when {

                expression {
                    env.HAS_CHANGES == 'true'
                }
            }

            steps {

                withCredentials(
                    [
                        usernamePassword(
                            credentialsId: 'github-creds',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]
                ) {

                    sh '''
                        set -e

                        echo "Configuring Git..."

                        git config user.name "Jenkins"

                        git config user.email "jenkins@example.com"

                        echo "Adding changed files..."

                        git add $FILES

                        if git diff --cached --quiet
                        then

                            echo "No staged changes."

                        else

                            echo "Creating commit..."

                            git commit \
                                -m "[skip ci] Jenkins: updated $FILES (build #${BUILD_NUMBER})"

                            echo "Pulling latest changes..."

                            git pull \
                                --rebase \
                                "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" \
                                "$BRANCH"

                            echo "Pushing changes..."

                            git push \
                                "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" \
                                "HEAD:${BRANCH}"

                            echo "Git push completed successfully."

                        fi
                    '''
                }
            }
        }

        // ----------------------------------------------------
        // 6. Maven Build
        // ----------------------------------------------------
        stage('Maven Build') {

            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo "JAVA VERSION"
                    echo "========================================"

                    java -version

                    echo "========================================"
                    echo "MAVEN VERSION"
                    echo "========================================"

                    mvn -version

                    echo "========================================"
                    echo "CHECKING POM.XML"
                    echo "========================================"

                    if [ ! -f pom.xml ]; then

                        echo "ERROR: pom.xml not found in repository root."

                        exit 1
                    fi

                    echo "pom.xml found."

                    echo "========================================"
                    echo "STARTING MAVEN BUILD"
                    echo "========================================"

                    mvn -B clean package

                    echo "========================================"
                    echo "MAVEN BUILD COMPLETED"
                    echo "========================================"
                '''
            }
        }

        // ----------------------------------------------------
        // 7. Check Artifact
        // ----------------------------------------------------
        stage('Check Artifact') {

            steps {

                sh '''
                    echo "========================================"
                    echo "TARGET DIRECTORY"
                    echo "========================================"

                    ls -lah target/

                    echo "========================================"
                    echo "GENERATED ARTIFACTS"
                    echo "========================================"

                    find target \
                        -maxdepth 1 \
                        -type f \
                        \\( -name "*.jar" -o -name "*.war" \\) \
                        -print
                '''
            }
        }

        // ----------------------------------------------------
        // 8. Archive Artifact
        // ----------------------------------------------------
        stage('Archive Artifact') {

            steps {

                archiveArtifacts(
                    artifacts: 'target/*.jar, target/*.war',
                    fingerprint: true,
                    allowEmptyArchive: false
                )
            }
        }
    }

    // ========================================================
    // POST ACTIONS
    // ========================================================
    post {

        success {

            script {

                if (env.HAS_CHANGES == 'true') {

                    echo "========================================"
                    echo "PIPELINE SUCCESS"
                    echo "========================================"

                    echo "Updated files : ${env.FILES}"

                    echo "GitHub push   : SUCCESS"

                    echo "Maven build   : SUCCESS"

                    echo "Artifact      : ARCHIVED"

                    echo "Repository    : https://${env.REPO.replace('.git', '')}"

                    echo "Branch        : ${env.BRANCH}"

                    echo "========================================"

                } else {

                    echo "========================================"
                    echo "PIPELINE SUCCESS"
                    echo "========================================"

                    echo "No JSON changes detected."

                    echo "GitHub push   : SKIPPED"

                    echo "Maven build   : SUCCESS"

                    echo "Artifact      : ARCHIVED"

                    echo "========================================"
                }
            }
        }

        failure {

            echo "========================================"
            echo "PIPELINE FAILED"
            echo "========================================"

            echo "Check Console Output."

            echo "Possible areas:"
            echo "1. Input validation"
            echo "2. JSON/jq processing"
            echo "3. Git checkout"
            echo "4. GitHub push"
            echo "5. Maven build"
            echo "6. Artifact generation"

            echo "========================================"
        }

        aborted {

            echo "========================================"
            echo "PIPELINE ABORTED"
            echo "========================================"

            echo "Required inputs were missing or the build was cancelled."

            echo "No invalid changes should be pushed."

            echo "========================================"
        }
    }
}
```
