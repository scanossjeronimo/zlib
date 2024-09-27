pipeline {
    parameters {
        // REPOSITORY variables
        string(name: 'GITHUB_TOKEN_ID', defaultValue: "gh-token", description: 'Github repository token credential id')

        // SCAN Variables
        string(name: 'SCANOSS_API_TOKEN_ID', defaultValue: "scanoss-token", description: 'The reference ID for the SCANOSS API TOKEN credential')
        string(name: 'SCANOSS_SBOM_IDENTIFY', defaultValue: "sbom.json", description: 'SCANOSS SBOM Identify filename')
        string(name: 'SCANOSS_SBOM_IGNORE', defaultValue: "sbom-ignore.json", description: 'SCANOSS SBOM Ignore filename')
        string(name: 'SCANOSS_CLI_DOCKER_IMAGE', defaultValue: "ghcr.io/scanoss/scanoss-py:latest", description: 'SCANOSS CLI Docker Image')
        booleanParam(name: 'ENABLE_DELTA_ANALYSIS', defaultValue: false, description: 'Analyze those files what have changed or new ones')
        booleanParam(name: 'MANUAL_TRIGGER', defaultValue: false, description: 'Trigger the pipeline manually')
    }
    agent any
    stages {
        stage('SCANOSS') {
            when {
                anyOf {
                    expression { 
                        echo "Manual Trigger: ${params.MANUAL_TRIGGER}"
                        return params.MANUAL_TRIGGER 
                    }
                    allOf {
                        expression { 
                            def payload = readJSON text: env.payload ?: '{}'
                            echo "Payload: ${payload}"
                            def result = payload.pull_request != null && 
                                   payload.pull_request.base.ref == 'main' && 
                                   payload.action == 'opened'
                            echo "Pull Request Condition: ${result}"
                            return result
                        }
                    }
                }
            }

            agent {
                docker {
                    image params.SCANOSS_CLI_DOCKER_IMAGE
                    args '--entrypoint='
                    reuseNode true
                }
            }
            steps {               
                script {
                    echo "SCANOSS stage is running"
                    
                    // File names
                    env.SCANOSS_RESULTS_JSON_FILE = "scanoss-results.json"
                    env.SCANOSS_LICENSE_CSV_FILE = "scanoss_license_data.csv"
                    env.SCANOSS_COPYLEFT_MD_FILE = "copyleft.md"

                    // Create Resources folder
                    env.SCANOSS_BUILD_BASE_PATH = "scanoss/${currentBuild.number}"
                    sh '''
                        mkdir -p ${SCANOSS_BUILD_BASE_PATH}/reports
                        mkdir -p ${SCANOSS_BUILD_BASE_PATH}/repository
                        mkdir -p ${SCANOSS_BUILD_BASE_PATH}/delta
                    '''
                    env.SCANOSS_REPORT_FOLDER_PATH = "${SCANOSS_BUILD_BASE_PATH}/reports"

                    // Resources Paths
                    env.SCANOSS_LICENSE_FILE_PATH = "${env.SCANOSS_REPORT_FOLDER_PATH}/${env.SCANOSS_LICENSE_CSV_FILE}"
                    env.SCANOSS_RESULTS_FILE_PATH = "${env.SCANOSS_REPORT_FOLDER_PATH}/${SCANOSS_RESULTS_JSON_FILE}"
                    env.SCANOSS_COPYLEFT_FILE_PATH = "${env.SCANOSS_REPORT_FOLDER_PATH}/${env.SCANOSS_COPYLEFT_MD_FILE}"

                    // Get Repository name and repo URL
                    if (params.MANUAL_TRIGGER) {
                        env.REPOSITORY_NAME = "integrations-jenkins"
                        env.REPOSITORY_URL = "https://github.com/scanoss/integrations-jenkins"
                    } else {
                        def payloadJson = readJSON text: env.payload
                        env.REPOSITORY_NAME = payloadJson.pull_request.base.repo.name
                        env.REPOSITORY_URL = payloadJson.pull_request.base.repo.html_url
                    }

                    // Checkout repository
                    dir("${env.SCANOSS_BUILD_BASE_PATH}/repository") {
                        git branch: 'main',
                            credentialsId: params.GITHUB_TOKEN_ID,
                            url: env.REPOSITORY_URL
                    }

                    // Delta Scan
                    deltaScan()

                    // Scan
                    env.SCAN_FOLDER = "${env.SCANOSS_BUILD_BASE_PATH}/" + (params.ENABLE_DELTA_ANALYSIS && !params.MANUAL_TRIGGER ? 'delta' : 'repository')
                    scan()

                    // Upload Artifacts
                    uploadArtifacts()

                    // Analyze results for copyleft
                    copyleft()

                    // Publish report on Jenkins dashboard
                    publishReport()
                }
            }
        }
    }
}

def publishReport() {
    publishReport name: "Scan Results", displayType: "dual", provider: csv(id: "report-summary", pattern: "${env.SCANOSS_LICENSE_FILE_PATH}")
}

def copyleft() {
    try {
        env.check_result = sh(returnStdout: true, script: '''
            echo 'component,name,copyleft' > $SCANOSS_LICENSE_FILE_PATH
            jq -r 'reduce .[]?[] as $item ({}; select($item.purl) | .[$item.purl[0] + "@" + $item.version] += [$item.licenses[]? | select(.copyleft == "yes") | .name]) | to_entries[] | select(.value | unique | length > 0) | [.key, .key, (.value | unique | length)] | @csv' $SCANOSS_RESULTS_FILE_PATH >> $SCANOSS_LICENSE_FILE_PATH
            printf '|| Component || Purl || Version || Licenses||\n' > $SCANOSS_COPYLEFT_FILE_PATH
            jq -r '[.[]?[] | select(.licenses) | select(.licenses[] | .copyleft? == "yes") | {component: .component, version: .version, purl: .purl[], licenses: (.licenses | map(.name) | join(","))}] | unique_by(.purl) | sort_by(.component) | to_entries[] | "|\\(.value.component)|\\(.value.purl)|\\(.value.version)|\\(.value.licenses)|"'  $SCANOSS_RESULTS_FILE_PATH >> $SCANOSS_COPYLEFT_FILE_PATH
            check_result=$(if [ $(wc -l < $SCANOSS_LICENSE_FILE_PATH) -gt 1 ]; then echo "1"; else echo "0"; fi);
            echo $check_result
        ''').trim()

        if (params.ABORT_ON_POLICY_FAILURE && env.check_result != '0') {
            currentBuild.result = "FAILURE"
        }
    } catch(e) {
        echo "Error in copyleft analysis: ${e.getMessage()}"
        if (params.ABORT_ON_POLICY_FAILURE) {
            currentBuild.result = "FAILURE"
        }
    }
}

def uploadArtifacts() {
    def scanossResultsPath = "${env.SCANOSS_RESULTS_FILE_PATH}"
    archiveArtifacts artifacts: scanossResultsPath, onlyIfSuccessful: true
}

def scan() {
    withCredentials([string(credentialsId: params.SCANOSS_API_TOKEN_ID, variable: 'SCANOSS_API_TOKEN')]) {
        dir("${env.SCAN_FOLDER}") {
            sh '''
                SBOM_IDENTIFY=""
                if [ -f $SCANOSS_SBOM_IDENTIFY ]; then SBOM_IDENTIFY="--identify $SCANOSS_SBOM_IDENTIFY" ; fi

                SBOM_IGNORE=""
                if [ -f $SCANOSS_SBOM_IGNORE ]; then SBOM_IGNORE="--ignore $SCANOSS_SBOM_IGNORE" ; fi

                CUSTOM_URL=""
                if [ ! -z $SCANOSS_API_URL ]; then CUSTOM_URL="--apiurl $SCANOSS_API_URL"; else CUSTOM_URL="--apiurl https://osskb.org/api/scan/direct" ; fi

                CUSTOM_TOKEN=""
                if [ ! -z $SCANOSS_API_TOKEN ]; then CUSTOM_TOKEN="--key $SCANOSS_API_TOKEN" ; fi

                scanoss-py scan $CUSTOM_URL $CUSTOM_TOKEN $SBOM_IDENTIFY $SBOM_IGNORE --output $WORKSPACE/$SCANOSS_REPORT_FOLDER_PATH/$SCANOSS_RESULTS_JSON_FILE .
            '''
        }
    }
}

def deltaScan() {
    if (params.ENABLE_DELTA_ANALYSIS && !params.MANUAL_TRIGGER) {
        echo 'Delta Scan Analysis enabled'
        try {
            def payloadJson = readJSON text: env.payload ?: '{}'
            def commits = payloadJson.commits ?: []
            def destinationFolder = "${env.SCANOSS_BUILD_BASE_PATH}/delta"
            def uniqueFileNames = new HashSet()

            commits.each { commit ->
                (commit.modified ?: []).each { fileName ->
                    uniqueFileNames.add(fileName.trim())
                }
                (commit.added ?: []).each { fileName ->
                    uniqueFileNames.add(fileName.trim())
                }
            }

            dir("${env.SCANOSS_BUILD_BASE_PATH}/repository") {
                uniqueFileNames.each { file ->
                    def sourcePath = "${file}"
                    def destinationPath = "${destinationFolder}"
                    sh "cp --parents ${sourcePath} ${destinationPath} || true"
                }
            }
        } catch (Exception e) {
            echo "Error in delta scan: ${e.message}"
            echo "Proceeding with full scan"
        }
    } else {
        echo 'Delta Scan Analysis disabled or manual trigger used'
    }
}
