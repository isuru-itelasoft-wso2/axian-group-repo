pipeline {

    agent any

    environment {
        APIM_ENV = "group"
        APICTL_HOME = "/opt/wso2/apictl"
        PATH = "/opt/wso2/apictl:${env.PATH}"

        /*
         * WSO2 API-M Publisher REST API
         */
        APIM_PUBLISHER_URL = "https://group-apim.example.com:9443"
    }

    stages {

        /*
         * ============================================================
         * 1. CHECK APICTL
         * ============================================================
         */
        stage('Check APICTL') {
            steps {
                sh '''
                    echo "========================================"
                    echo "CHECKING APICTL"
                    echo "========================================"

                    echo "USER=$(whoami)"
                    echo "HOME=$HOME"
                    echo "APICTL_HOME=$APICTL_HOME"
                    echo "PATH=$PATH"

                    echo ""
                    echo "APICTL LOCATION:"
                    which apictl

                    echo ""
                    echo "APICTL VERSION:"
                    apictl version

                    echo "========================================"
                '''
            }
        }


        /*
         * ============================================================
         * 2. CHECKOUT SOURCE CODE
         * ============================================================
         */
        stage('Checkout') {
            steps {

                checkout scm

                sh '''
                    echo "========================================"
                    echo "CHECKOUT COMPLETED"
                    echo "========================================"

                    echo "Git branch:"
                    git branch --show-current || true

                    echo ""
                    echo "Latest commit:"
                    git log -1 --oneline

                    echo ""
                    echo "Repository contents:"
                    ls -la

                    echo "========================================"
                '''
            }
        }


        /*
         * ============================================================
         * 3. DETECT CHANGED APIs
         * ============================================================
         */
        stage('Detect Changed APIs') {
            steps {

                script {

                    echo "========================================"
                    echo "DETECTING CHANGED APIs"
                    echo "========================================"

                    /*
                     * Determine previous commit.
                     */
                    def changedFiles = sh(
                        script: '''
                            git diff --name-only HEAD~1 HEAD
                        ''',
                        returnStdout: true
                    ).trim()


                    echo ""
                    echo "Changed files:"
                    echo "----------------------------------------"

                    if (changedFiles) {
                        echo changedFiles
                    } else {
                        echo "No changed files detected."
                    }

                    echo "----------------------------------------"


                    /*
                     * Extract API project names.
                     */
                    def apiProjects = []

                    if (changedFiles) {

                        changedFiles.split("\\n").each { file ->

                            file = file.trim()

                            if (!file) {
                                return
                            }


                            /*
                             * Ignore repository-level files.
                             */
                            if (!file.contains("/")) {

                                echo "Ignoring repository-level file: ${file}"

                                return
                            }


                            /*
                             * Extract first directory.
                             */
                            def parts = file.split("/")

                            if (parts.length > 1) {

                                def apiProject = parts[0]


                                /*
                                 * Ignore common non-API directories.
                                 */
                                if (
                                    apiProject == ".git" ||
                                    apiProject == "governance" ||
                                    apiProject == "scripts" ||
                                    apiProject == "config"
                                ) {

                                    echo "Ignoring non-API directory: ${apiProject}"

                                    return
                                }


                                /*
                                 * Verify directory exists.
                                 */
                                if (!fileExists(apiProject)) {

                                    echo "Ignoring unknown directory: ${apiProject}"

                                    return
                                }


                                /*
                                 * Avoid duplicates.
                                 */
                                if (!apiProjects.contains(apiProject)) {

                                    apiProjects.add(apiProject)
                                }
                            }
                        }
                    }


                    /*
                     * Fail if no API changed.
                     */
                    if (apiProjects.isEmpty()) {

                        echo ""
                        echo "========================================"
                        echo "NO API CHANGES DETECTED"
                        echo "========================================"

                        echo "No API will be deployed."

                        echo "========================================"

                        error(
                            "No API project was changed in this commit. " +
                            "Pipeline stopped."
                        )
                    }


                    /*
                     * Sort APIs.
                     */
                    apiProjects.sort()


                    /*
                     * Store list for subsequent stages.
                     */
                    env.API_PROJECTS = apiProjects.join(",")


                    echo ""
                    echo "========================================"
                    echo "APIs TO PROCESS"
                    echo "========================================"

                    apiProjects.each { apiProject ->

                        echo "  -> ${apiProject}"
                    }

                    echo ""
                    echo "Total APIs to process: ${apiProjects.size()}"

                    echo "========================================"
                }
            }
        }


        /*
         * ============================================================
         * 4. LOGIN TO TIGO APIM
         * ============================================================
         */
        stage('Login to APIM') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'cicd-apictl',
                        usernameVariable: 'APIM_USER',
                        passwordVariable: 'APIM_PASS'
                    )
                ]) {

                    sh '''
                        echo "========================================"
                        echo "LOGIN TO APIM"
                        echo "========================================"

                        printf "%s\\n" "$APIM_PASS" | \
                        apictl login "${APIM_ENV}" \
                            -u "$APIM_USER" \
                            --password-stdin \
                            --insecure

                        echo ""
                        echo "Successfully logged in to APIM environment:"
                        echo "${APIM_ENV}"

                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * ============================================================
         * 5. GOVERNANCE GATE
         * ============================================================
         */
        stage('Governance Gate') {
            steps {

                script {

                    def apiProjects = env.API_PROJECTS.split(",")

                    def failedApis = []

                    def passedApis = []


                    echo ""
                    echo "========================================"
                    echo "GOVERNANCE GATE"
                    echo "========================================"


                    apiProjects.each { apiProject ->

                        echo ""
                        echo "----------------------------------------"
                        echo "GOVERNANCE CHECK: ${apiProject}"
                        echo "----------------------------------------"


                        /*
                         * Make sure API project exists.
                         */
                        if (!fileExists(apiProject)) {

                            echo "ERROR: API project not found: ${apiProject}"

                            failedApis.add(apiProject)

                            return
                        }


                        /*
                         * Run APIM dry-run validation.
                         */
                        def governanceOutput = sh(
                            script: """
                                set +e

                                apictl import api \\
                                    --file "${apiProject}" \\
                                    --environment "${APIM_ENV}" \\
                                    --dry-run \\
                                    --insecure

                                exit 0
                            """,
                            returnStdout: true
                        ).trim()


                        echo ""
                        echo "========== GOVERNANCE RESULT =========="

                        echo governanceOutput

                        echo "========================================"


                        /*
                         * Count ERROR occurrences.
                         */
                        def errorCount = governanceOutput.count("ERROR")


                        echo ""
                        echo "Governance ERROR count for ${apiProject}: ${errorCount}"


                        if (errorCount > 0) {

                            echo ""
                            echo "❌ GOVERNANCE FAILED: ${apiProject}"

                            failedApis.add(apiProject)

                        } else {

                            echo ""
                            echo "✅ GOVERNANCE PASSED: ${apiProject}"

                            passedApis.add(apiProject)
                        }
                    }


                    /*
                     * Print governance summary.
                     */
                    echo ""
                    echo "========================================"
                    echo "GOVERNANCE SUMMARY"
                    echo "========================================"


                    echo ""
                    echo "PASSED APIs:"

                    if (passedApis.isEmpty()) {

                        echo "  None"

                    } else {

                        passedApis.each {

                            echo "  ✅ ${it}"
                        }
                    }


                    echo ""
                    echo "FAILED APIs:"

                    if (failedApis.isEmpty()) {

                        echo "  None"

                    } else {

                        failedApis.each {

                            echo "  ❌ ${it}"
                        }
                    }


                    echo ""
                    echo "========================================"


                    /*
                     * Block deployment if ANY API failed.
                     */
                    if (!failedApis.isEmpty()) {

                        error(
                            "GOVERNANCE GATE FAILED. " +
                            "The following API(s) failed governance validation: " +
                            failedApis.join(", ") +
                            ". NO APIs will be deployed."
                        )
                    }


                    echo ""
                    echo "========================================"
                    echo "ALL GOVERNANCE CHECKS PASSED"
                    echo "========================================"
                }
            }
        }


        /*
         * ============================================================
         * 6. DEPLOY CHANGED APIs
         * ============================================================
         */
        stage('Deploy Changed APIs') {
            steps {

                script {

                    def apiProjects = env.API_PROJECTS.split(",")


                    echo ""
                    echo "========================================"
                    echo "DEPLOYING CHANGED APIs"
                    echo "========================================"


                    apiProjects.each { apiProject ->

                        echo ""
                        echo "----------------------------------------"
                        echo "DEPLOYING: ${apiProject}"
                        echo "----------------------------------------"


                        sh """
                            apictl import api \\
                                --file "${apiProject}" \\
                                --environment "${APIM_ENV}" \\
                                --update \\
                                --insecure
                        """


                        echo ""
                        echo "✅ Deployment completed: ${apiProject}"
                    }


                    echo ""
                    echo "========================================"
                    echo "ALL CHANGED APIs DEPLOYED"
                    echo "========================================"
                }
            }
        }


        /*
         * ============================================================
         * 7. CONFIGURE API MONETIZATION
         *
         * For every deployed API:
         *
         * 1. Read API name from api.yaml
         * 2. Find API ID using Publisher REST API
         * 3. Check monetization status
         * 4. If already enabled -> skip
         * 5. If disabled -> enable monetization
         * 6. Set ConnectedAccountKey
         *
         * ============================================================
         */
        stage('Configure Monetization') {

            steps {

                script {

                    def apiProjects = env.API_PROJECTS.split(",")


                    echo ""
                    echo "========================================"
                    echo "CONFIGURING API MONETIZATION"
                    echo "========================================"


                    /*
                     * APIM credentials
                     *
                     * Stripe ConnectedAccountKey should be stored
                     * as a Jenkins Secret Text credential.
                     */
                    withCredentials([

                        usernamePassword(
                            credentialsId: 'cicd-apictl',
                            usernameVariable: 'APIM_USER',
                            passwordVariable: 'APIM_PASS'
                        ),

                        string(
                            credentialsId: 'stripe-connected-account-key',
                            variable: 'CONNECTED_ACCOUNT_KEY'
                        )

                    ]) {


                        apiProjects.each { apiProject ->


                            echo ""
                            echo "----------------------------------------"
                            echo "MONETIZATION: ${apiProject}"
                            echo "----------------------------------------"


                            /*
                             * ==================================================
                             * 1. READ API NAME FROM api.yaml
                             * ==================================================
                             *
                             * Expected:
                             *
                             * data:
                             *   name: Charging API
                             *
                             */
                            def apiName = sh(

                                script: """

                                    python3 - <<'PY'

import yaml

with open("${apiProject}/api.yaml", "r") as f:

    data = yaml.safe_load(f)

api_data = data.get("data", data)

api_name = api_data.get("name", "")

print(api_name)

PY

                                """,

                                returnStdout: true

                            ).trim()


                            if (!apiName) {

                                error(
                                    "Unable to determine API name from " +
                                    "${apiProject}/api.yaml"
                                )
                            }


                            echo "API Name: ${apiName}"


                            /*
                             * ==================================================
                             * 2. FIND API ID
                             * ==================================================
                             *
                             * GET
                             *
                             * /api/am/publisher/v4/apis
                             *
                             * ?query=name:<API_NAME>
                             *
                             */
                            def apiSearchFile =
                                "${apiProject}/api-search-response.json"


                            sh """

                                curl -ksS \\
                                    -u "\$APIM_USER:\$APIM_PASS" \\
                                    --get \\
                                    --data-urlencode "query=name:${apiName}" \\
                                    "${APIM_PUBLISHER_URL}/api/am/publisher/v4/apis" \\
                                    -o "${apiSearchFile}"

                            """


                            /*
                             * Extract API ID.
                             */
                            def apiId = sh(

                                script: """

                                    python3 - <<'PY'

import json

with open("${apiSearchFile}", "r") as f:

    data = json.load(f)

apis = data.get("list", [])

if not apis:

    raise SystemExit(1)

print(apis[0].get("id", ""))

PY

                                """,

                                returnStdout: true

                            ).trim()


                            if (!apiId) {

                                error(
                                    "API ID could not be found for API: " +
                                    "${apiName}"
                                )
                            }


                            echo "API ID: ${apiId}"


                            /*
                             * ==================================================
                             * 3. CHECK MONETIZATION STATUS
                             * ==================================================
                             *
                             * GET
                             *
                             * /apis/{api_id}/monetization
                             */
                            def monetizationFile =
                                "${apiProject}/monetization-response.json"


                            sh """

                                curl -ksS \\
                                    -u "\$APIM_USER:\$APIM_PASS" \\
                                    "${APIM_PUBLISHER_URL}/api/am/publisher/v4/apis/${apiId}/monetization" \\
                                    -o "${monetizationFile}"

                            """


                            echo ""
                            echo "Monetization response:"
                            echo "----------------------------------------"

                            sh """

                                cat "${monetizationFile}"

                            """

                            echo "----------------------------------------"


                            /*
                             * Extract enabled flag.
                             */
                            def monetizationEnabled = sh(

                                script: """

                                    python3 - <<'PY'

import json

with open("${monetizationFile}", "r") as f:

    data = json.load(f)

print(str(data.get("enabled", False)).lower())

PY

                                """,

                                returnStdout: true

                            ).trim()


                            echo ""
                            echo "Monetization enabled: ${monetizationEnabled}"


                            /*
                             * ==================================================
                             * 4. ENABLE MONETIZATION IF REQUIRED
                             * ==================================================
                             */
                            if (monetizationEnabled == "true") {


                                echo ""
                                echo "✅ Monetization already enabled"
                                echo "Skipping monetization configuration."


                            } else {


                                echo ""
                                echo "Monetization is NOT enabled."
                                echo "Enabling monetization..."


                                /*
                                 * ==================================================
                                 * 5. CREATE MONETIZATION PAYLOAD
                                 * ==================================================
                                 *
                                 * ConnectedAccountKey comes from Jenkins
                                 * Secret Text credential.
                                 */
                                def monetizationPayload =
                                    """{
  "enabled": true,
  "properties": {
    "ConnectedAccountKey": "${env.CONNECTED_ACCOUNT_KEY}"
  }
}"""


                                writeFile(

                                    file:
                                        "${apiProject}/monetization.json",

                                    text:
                                        monetizationPayload

                                )


                                /*
                                 * ==================================================
                                 * 6. POST /monetize
                                 * ==================================================
                                 */
                                sh """

                                    curl -ksS --fail-with-body \\
                                        -u "\$APIM_USER:\$APIM_PASS" \\
                                        -X POST \\
                                        -H "Content-Type: application/json" \\
                                        --data-binary "@${apiProject}/monetization.json" \\
                                        "${APIM_PUBLISHER_URL}/api/am/publisher/v4/apis/${apiId}/monetize"

                                """


                                echo ""
                                echo "✅ Monetization enabled"
                                echo "API: ${apiName}"
                                echo "API ID: ${apiId}"

                            }


                            /*
                             * ==================================================
                             * 7. CLEAN TEMPORARY FILES
                             * ==================================================
                             */
                            sh """

                                rm -f \
                                    "${apiProject}/api-search-response.json" \
                                    "${apiProject}/monetization-response.json" \
                                    "${apiProject}/monetization.json"

                            """


                            echo ""
                            echo "----------------------------------------"
                            echo "MONETIZATION COMPLETED: ${apiProject}"
                            echo "----------------------------------------"
                        }
                    }


                    echo ""
                    echo "========================================"
                    echo "ALL MONETIZATION CONFIGURATION COMPLETED"
                    echo "========================================"
                }
            }
        }
    }


    /*
     * ================================================================
     * POST BUILD
     * ================================================================
     */
    post {

        success {

            echo ""
            echo "========================================"
            echo "       API CI/CD SUCCESS"
            echo "========================================"


            echo ""
            echo "APIs processed:"

            script {

                if (env.API_PROJECTS) {

                    echo "${env.API_PROJECTS}"

                } else {

                    echo "None"
                }
            }


            echo ""
            echo "Governance: PASSED"
            echo "Deployment: COMPLETED"
            echo "Monetization: CONFIGURED"

            echo ""
            echo "Only changed APIs were deployed."

            echo "========================================"
        }


        failure {

            echo ""
            echo "========================================"
            echo "       API CI/CD FAILED"
            echo "========================================"


            echo ""
            echo "APIs detected:"

            script {

                if (env.API_PROJECTS) {

                    echo "${env.API_PROJECTS}"

                } else {

                    echo "No API list available."
                }
            }


            echo ""
            echo "Governance / Deployment / Monetization: FAILED"
            echo "API deployment or monetization configuration failed."

            echo ""
            echo "========================================"
        }
    }
}