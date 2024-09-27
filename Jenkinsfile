pipeline {
    parameters {
        // SCAN Variables
        string(name: 'SCANOSS_API_TOKEN_ID', defaultValue: "scanoss-token", description: 'The reference ID for the SCANOSS API TOKEN credential')
        string(name: 'SCANOSS_SBOM_IDENTIFY', defaultValue: "sbom.json", description: 'SCANOSS SBOM Identify filename')
        string(name: 'SCANOSS_SBOM_IGNORE', defaultValue: "sbom-ignore.json", description: 'SCANOSS SBOM Ignore filename')
        string(name: 'SCANOSS_CLI_DOCKER_IMAGE', defaultValue: "ghcr.io/scanoss/scanoss-py:latest", description: 'SCANOSS CLI Docker Image')
        booleanParam(name: 'ENABLE_DELTA_ANALYSIS', defaultValue: false, description: 'Analyze those files what have changed or new ones')
    }
    agent any
    stages {
        stage('SCANOSS') {
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
                    env.REPOSITORY_NAME = "zlib"  // Replace with your actual repository name
                    env.REPOSITORY_URL = "https://github.com/scanossjeronimo/zlib"  // Replace with your actual repository URL

                    echo "Repository Name: ${env.REPOSITORY_NAME}"
                    echo "Repository URL: ${env.REPOSITORY_URL}"

                    // Checkout repository
                    dir("${env.SCANOSS_BUILD_BASE_PATH}/repository") {
                        git branch: 'main',
                            url: env.REPOSITORY_URL
                    }

                    // Scan
                    env.SCAN_FOLDER = "${env.SCANOSS_BUILD_BASE_PATH}/repository"
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

// ... rest of your functions (publishReport, copyleft, uploadArtifacts, scan) ...
