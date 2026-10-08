// ============================================================
// Pipeline 2 : node/ directory JSON files update
// Files   : dev.json, prod.json, stage.json, uat.json (tick chesina files marutayi)
// Rule    : Anni fields ki value ivvali. Oka field empty unna build ABORT avutundi.
// Auto    : Manual approval ledu. Validation pass ayite automatic ga push avutundi.
//           File tick cheyakapoyina / ye field empty unna build automatic ga ABORT avutundi.
// ============================================================

// Build ni FAILED kakunda ABORTED ga automatic ga aapadaniki helper
def abortBuild(String msg) {
    currentBuild.result = 'ABORTED'
    error(msg)
}

pipeline {
    agent any

    parameters {
        // ---- E files marchali ----
        booleanParam(name: 'DEV_JSON',   defaultValue: false, description: 'node/dev.json')
        booleanParam(name: 'PROD_JSON',  defaultValue: false, description: 'node/prod.json')
        booleanParam(name: 'STAGE_JSON', defaultValue: false, description: 'node/stage.json')
        booleanParam(name: 'UAT_JSON',   defaultValue: false, description: 'node/uat.json')

        // ---- Values ----
        string(name: 'P_ENVIRONMENT',   defaultValue: '', description: '"environment" (e.g. dev)')
        string(name: 'P_NODE_NAME',     defaultValue: '', description: '"nodeName" (e.g. node-02)')
        string(name: 'P_NODE_TYPE',     defaultValue: '', description: '"nodeType" (e.g. worker)')
        string(name: 'P_REGION',        defaultValue: '', description: '"region" (e.g. us-east-1)')
        string(name: 'P_AZ',            defaultValue: '', description: '"availabilityZone" (e.g. us-east-1b)')
        string(name: 'P_INSTANCE_TYPE', defaultValue: '', description: '"instanceType" (e.g. t3.large)')
        string(name: 'P_OS',            defaultValue: '', description: '"os" (e.g. Amazon Linux 2023)')
        string(name: 'P_K8S_ROLE',      defaultValue: '', description: '"kubernetes.role" (e.g. worker)')
        string(name: 'P_K8S_VERSION',   defaultValue: '', description: '"kubernetes.version" (e.g. 1.36)')
        string(name: 'P_CPU',           defaultValue: '', description: '"resources.cpu" (e.g. 2)')
        string(name: 'P_MEMORY',        defaultValue: '', description: '"resources.memory" (e.g. 4Gi)')
        string(name: 'P_DISK',          defaultValue: '', description: '"resources.disk" (e.g. 50Gi)')
        string(name: 'P_LABEL_ENV',     defaultValue: '', description: '"labels.environment" (e.g. dev)')
        string(name: 'P_LABEL_TEAM',    defaultValue: '', description: '"labels.team" (e.g. platform)')
    }

    environment {
        REPO   = 'github.com/Rajesh33-11/flipkart.git'
        BRANCH = 'main'
    }

    stages {
        stage('Validate Inputs') {
            steps {
                script {
                    // 1) File select chesaara?
                    def files = []
                    if (params.DEV_JSON)   { files << 'node/dev.json' }
                    if (params.PROD_JSON)  { files << 'node/prod.json' }
                    if (params.STAGE_JSON) { files << 'node/stage.json' }
                    if (params.UAT_JSON)   { files << 'node/uat.json' }
                    if (files.isEmpty()) {
                        abortBuild('ABORTED: Kaneesam okka file (dev/prod/stage/uat) tick cheyandi')
                    }
                    env.FILES = files.join(' ')

                    // 2) Entered values list
                    def valueNames = ['P_ENVIRONMENT', 'P_NODE_NAME', 'P_NODE_TYPE', 'P_REGION', 'P_AZ', 'P_INSTANCE_TYPE', 'P_OS', 'P_K8S_ROLE', 'P_K8S_VERSION', 'P_CPU', 'P_MEMORY', 'P_DISK', 'P_LABEL_ENV', 'P_LABEL_TEAM']
                    def entered = []
                    def missing = []
                    valueNames.each { n ->
                        def v = params[n]
                        if (v == null || v.toString().trim() == '' || v.toString().trim() == 'no-change') {
                            missing << n
                        } else {
                            entered << "${n}=${v}"
                        }
                    }
                    // Oka value kooda miss ayina (empty unna) build ABORT avutundi
                    if (!missing.isEmpty()) {
                        abortBuild("ABORTED: Ee fields ki value ivvaledu: ${missing.join(', ')}. Anni fields ki value ivvali.")
                    }

                    // 3) Number fields number ayyi undali
                    def numericNames = []
                    numericNames.each { n ->
                        def v = params[n]
                        if (v != null && v.toString().trim() != '' && !v.toString().matches('[0-9]+')) {
                            abortBuild("ABORTED: ${n} number ayyi undali, meeru icchindi: '${v}'")
                        }
                    }

                    // 4) Jenkins build list lo kanipinchedi (verify cheyadaniki)
                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${files.join(', ')}"
                    currentBuild.description = "Files: ${files.join(', ')} | Changes: ${entered.join(', ')}"

                    echo "Files   : ${env.FILES}"
                    echo "Changes : ${entered.join(', ')}"
                }
            }
        }

        stage('Checkout') {
            steps {
                cleanWs()
                git branch: "${env.BRANCH}", url: "https://${env.REPO}", credentialsId: 'github-creds'
            }
        }

        stage('Update JSON') {
            steps {
                sh '''
                    set -e
                    cat > filter.jq <<'JQ'
def p($k): ($ENV[$k] // "");
def setstr($path; $k): if p($k) != "" then setpath($path; p($k)) else . end;
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
                    for f in $FILES; do
                        [ -f "$f" ] || { echo "File dorakaledu: $f"; exit 1; }
                        jq -f filter.jq "$f" > tmp.json
                        mv tmp.json "$f"
                        echo "=== Updated $f ==="
                        cat "$f"
                    done
                    rm -f filter.jq
                '''
            }
        }

        stage('Review Changes') {
            steps {
                script {
                    def rc = sh(returnStatus: true, script: 'git diff --quiet')
                    env.HAS_CHANGES = (rc != 0) ? 'true' : 'false'
                }
                sh '''
                    echo "================ GIT DIFF (verify chesukondi) ================"
                    git --no-pager diff --stat
                    git --no-pager diff
                '''
                script {
                    if (env.HAS_CHANGES == 'false') {
                        echo 'Marpulu levu: icchina values already JSON lo unnayi. Push avasaram ledu.'
                    }
                }
            }
        }

        stage('Commit & Push') {
            when { expression { env.HAS_CHANGES == 'true' } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds',
                                 usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git config user.name  "Jenkins"
                        git config user.email "jenkins@example.com"
                        git add $FILES
                        if git diff --cached --quiet; then
                            echo "Marpulu levu, commit avasaram ledu"
                        else
                            git commit -m "[skip ci] Jenkins: updated $FILES (build #${BUILD_NUMBER})"
                            git pull --rebase "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" "$BRANCH"
                            git push "https://${GIT_USER}:${GIT_TOKEN}@${REPO}" "HEAD:${BRANCH}"
                        fi
                    '''
                }
            }
        }

        // GitHub push ayyaka (updated JSON files tho) Maven artifact create chestundi
        stage('Maven Build') {
            steps {
                sh '''
                    set -e
                    [ -f pom.xml ] || { echo "pom.xml dorakaledu (repo root lo undali)"; exit 1; }
                    mvn -B clean package
                '''
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar, target/*.war', fingerprint: true, allowEmptyArchive: false
            }
        }
    }

    post {
        success {
            script {
                if (env.HAS_CHANGES == 'true') {
                    echo "SUCCESS: ${env.FILES} GitHub lo push ayyayi + Maven artifact create ayyindi. Verify: https://${env.REPO.replace('.git', '')}/commits/${env.BRANCH}"
                } else {
                    echo 'SUCCESS: Marpulu levu, push avvaledu. Maven artifact create ayyindi.'
                }
            }
        }
        failure {
            echo 'FAILED: Console Output chudandi (Validate / jq / git push / Maven errors).'
        }
        aborted {
            echo 'ABORTED: Inputs ivvaledu (file / values) leda cancel chesaru. Emi push avvaledu.'
        }
    }
}
