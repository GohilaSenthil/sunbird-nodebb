@Library('central-pipeline-lib@main') _
if (env.CHANGE_ID) {
    prValidationPipeline()
} else {
    postMergePipeline()
}
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

            stage('Build') {
                env.NODE_ENV = "build"
                sh("git clone https://github.com/NodeBB/NodeBB.git -b ${params.nodebb_branch}")
                sh("cp Dockerfile NodeBB")
                sh("cp build.sh NodeBB")
                print "Environment will be : ${env.NODE_ENV}"
                sh('chmod 777 NodeBB/build.sh')
                sh("bash ./NodeBB/build.sh ${build_tag}_${params.nodebb_branch} ${env.NODE_NAME} ${hub_org}")
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
