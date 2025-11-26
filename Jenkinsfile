node('build-slave') {
    try {
        String ANSI_GREEN = "\u001B[32m"
        String ANSI_NORMAL = "\u001B[0m"
        String ANSI_BOLD = "\u001B[1m"
        String ANSI_RED = "\u001B[31m"
        String ANSI_YELLOW = "\u001B[33m"

        ansiColor('xterm') {
            stage('Checkout') {
                if (!env.hub_org) {
                    println(ANSI_BOLD + ANSI_RED + "Uh Oh! Please set a Jenkins environment variable named hub_org with value as registery/sunbidrded" + ANSI_NORMAL)
                    error 'Please resolve the errors and rerun..'
                }
                else
                    println(ANSI_BOLD + ANSI_GREEN + "Found environment variable named hub_org with value as: " + hub_org + ANSI_NORMAL)
                cleanWs()
                def scmVars = checkout scm
                checkout scm: [$class: 'GitSCM', branches: [[name: "$params.github_release_tag"]], userRemoteConfigs: [[url: scmVars.GIT_URL]]]
                // We've to get the evaluated value of ${public_repo_branch}
                build_tag = sh(script: "echo "+params.github_release_tag.split('/')[-1] + "_" + env.BUILD_NUMBER, returnStdout: true).trim()
                println(ANSI_BOLD + ANSI_YELLOW + "github_release_tag specified, building from github_release_tag: " + params.github_release_tag + ANSI_NORMAL)
                echo "build_tag: " + build_tag
            }

if (params.enable_owasp_scan) {
    stage('Dependency Check (Pre-Build)') {
        script {
            def projectName = "${env.JOB_BASE_NAME}"
            def reportDir = "/var/lib/jenkins/owasp-report"
            def reportFile = "${reportDir}/${projectName}-owasp-report.html"
            def workspaceReportDir = "${env.WORKSPACE}/owasp-report"
            def workspaceReportFile = "${workspaceReportDir}/${projectName}-owasp-report.html"

            echo "🔍 Starting OWASP Dependency-Check for project: ${projectName}"
            echo "📁 Shared report path: ${reportFile}"
            echo "📁 Workspace report path: ${workspaceReportFile}"

            withEnv(["JAVA_HOME=${JAVA17_HOME}", "PATH=${JAVA17_HOME}/bin:${env.PATH}"]) {
                sh """
                    set -e
                    echo "✅ Java version in use:"
                    java -version

                    echo "🧭 Ensure shared report directory exists and is writable:"
                    mkdir -p ${reportDir}
                    ls -ld ${reportDir} || true

                    echo "🚀 Running OWASP Dependency-Check (writing to ${reportDir})..."
                    # run from workspace so --scan . is correct
                    cd "${env.WORKSPACE}"
                    time /var/lib/jenkins/dependency-check/bin/dependency-check.sh \
                        --project "${projectName}" \
                        --scan . \
                        --out ${reportDir} \
                        --format "HTML" \
                        --disableNodeAudit --disableRetireJS --disableAssembly

                    echo "📄 Attempt to move/rename generated report to job-specific name (if default created):"
                    # Default name dependency-check-report.html may be created in reportDir
                    if [ -f "${reportDir}/dependency-check-report.html" ]; then
                        mv -f "${reportDir}/dependency-check-report.html" "${reportFile}"
                    fi

                    echo "🔎 Verify report file exists and show details:"
                    if [ -f "${reportFile}" ]; then
                        ls -l "${reportFile}"
                    else
                        echo "!!! Report file NOT found at ${reportFile} !!!"
                        echo "Listing ${reportDir} contents for debugging:"
                        ls -la "${reportDir}" || true
                        # Fail here so pipeline shows clear error (optional). Comment next line if you prefer to continue.
                        exit 1
                    fi

                    echo "📋 Copying report into workspace for archiving..."
                    mkdir -p "${workspaceReportDir}"
                    cp -f "${reportFile}" "${workspaceReportFile}"
                    ls -l "${workspaceReportFile}"
                """
            }

            // Archive only this job's specific HTML report (workspace-relative)
            archiveArtifacts artifacts: "owasp-report/${projectName}-owasp-report.html", fingerprint: true
        }
    }
}                       

            stage('Build') {
                env.NODE_ENV = "build"
                sh("git clone https://github.com/NodeBB/NodeBB.git -b ${params.nodebb_branch}")
                sh("cp Dockerfile NodeBB")
                sh("cp build.sh NodeBB")
                print "Environment will be : ${env.NODE_ENV}"
                sh('chmod 777 NodeBB/build.sh')
                sh("bash ./NodeBB/build.sh ${build_tag}_${params.nodebb_branch} ${env.NODE_NAME} ${hub_org}")
            }

            // 🧩 New Stage: Docker Vulnerability Scan
            if (params.enable_docker_scan) {
                stage('Docker Scan') {
                    script {
                        def imageFullName = "${hub_org}/${JOB_BASE_NAME}:${build_tag}_${params.nodebb_branch}"
                        echo "🔍 Starting Trivy scan for image: ${imageFullName}"

                        sh """
                            mkdir -p trivy-reports

                            # Full Trivy scan in JSON format
                            trivy image --quiet --format json --output trivy-reports/trivy-report.json ${imageFullName}

                            # Extract by severity
                            jq '.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL")' trivy-reports/trivy-report.json > trivy-reports/critical.json || true
                            jq '.Results[].Vulnerabilities[] | select(.Severity=="HIGH")' trivy-reports/trivy-report.json > trivy-reports/high.json || true
                            jq '.Results[].Vulnerabilities[] | select(.Severity=="MEDIUM")' trivy-reports/trivy-report.json > trivy-reports/medium.json || true

                            echo "================== TRIVY VULNERABILITY SUMMARY =================="
                            jq -r '.Results[].Vulnerabilities[].Severity' trivy-reports/trivy-report.json | sort | uniq -c | awk '{print \$2": "\$1}'
                            echo "================================================================="
                        """

                        // Archive the scan reports
                        archiveArtifacts artifacts: 'trivy-reports/*.json', fingerprint: true
                    }
                }
            }

            stage('ArchiveArtifacts') {
                archiveArtifacts "metadata.json"
                currentBuild.description = "${build_tag}"
            }
        }

    }
    catch (err) {
        currentBuild.result = "FAILURE"
        throw err
    }

}
