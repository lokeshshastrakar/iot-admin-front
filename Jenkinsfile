#!groovy

node {

    projectName = "iot-admin-front"

    def sanitizedBuildTag = sh script: "echo -n '${env.BUILD_TAG}' | sed -r 's/[^a-zA-Z0-9_.-]//g' | tr '[:upper:]' '[:lower:]'", returnStdout: true
    imageName = "${projectName}-${sanitizedBuildTag}"
    buildImage = "${projectName}-build-${sanitizedBuildTag}"
    version = null
    dockerImageVersion = null
    dockerImageLatest = null
    jeanMichelAbortBuild = false
    
    try {

        stage('Checkout') {
            checkout scm

            // Jean-Michel Enattendant
            // Pour l'instant Jenkins n'ignore pas les messages de commit chore(release) du à un bug Jenkins, en attendant on fait une chinoiserie.
            // De ce fait on aura 2 builds successifs avec le second en erreur, mais on ne bouclera pas.
            // https://issues.jenkins-ci.org/browse/JENKINS-36836
            // https://issues.jenkins-ci.org/browse/JENKINS-36195
            def lastCommitMessage = sh script: 'git log -1 --pretty=oneline', returnStdout: true
            if (lastCommitMessage =~ /chore\(release\):/) {
                jeanMichelAbortBuild = true
                sh "echo 'Last commit is from Jenkins release, cancel execution'"
                sh "exit 1"
            }
        }

        stage('Build project') {
            docker.build("${buildImage}", "-f Dockerfile.build .")
            sh "docker run --rm -v \$(pwd | sed 's|/root/workspace/|/home/denouche/volumes/jenkins/workspace-tmp/|'):/usr/src/app/ ${buildImage} bash -c 'make clean getdeps && grunt build --mode=prod'"
            
            docker.build("${imageName}", '.')
        }

        if (isRelease()) {
            stage('Release') {
                releaseImageName = "${projectName}-release"
                docker.build(releaseImageName, "-f Dockerfile.release .")
                sh "pwd && ls -al && ls -al dist"
                sshagent (['6394728b-d88f-4534-b168-a513d8e6345b']) {
                    sh "docker run --rm -v /home/denouche/volumes/jenkins-agents/.ssh/id_rsa:/root/.ssh/id_rsa -v \$(pwd | sed 's|/root/workspace/|/home/denouche/volumes/jenkins/workspace-tmp/|'):/usr/src/app/ ${releaseImageName} bash -c 'make release'"
                }
                sh "docker rmi ${releaseImageName}"

                docker.build("${imageName}", '.')

                // Now the release is done if needed, retrieve version number
                version = getNPMVersion(readFile('package.json'))
                dockerImageVersion = "denouche/${projectName}:${version}"
                dockerImageLatest = "denouche/${projectName}:latest"
                sh "docker tag ${imageName} ${dockerImageVersion}"
                sh "docker tag ${imageName} ${dockerImageLatest}"
                docker.withRegistry('https://index.docker.io/v1/', 'ceba80c5-ac8d-407a-a43e-61dc1177b277') {
                    sh "docker push ${dockerImageVersion}"
                    sh "docker push ${dockerImageLatest}"
                }

                // Push the commit and the git tag only if docker image was successfully pushed
                sshagent (['6394728b-d88f-4534-b168-a513d8e6345b']) {
                    sh "git remote -v"
                    sh "git rm -rf dist && git commit -m'chore(release): clean dist folder after release'"
                    sh "git push --follow-tags origin HEAD"
                }
            }
        }

    }
    finally {
        if(jeanMichelAbortBuild) {
            currentBuild.result = 'ABORTED'
        }

        stage('Clean docker images') {
            sh "docker rmi ${buildImage} ${imageName} ${dockerImageVersion} ${dockerImageLatest} || true"
        }
    }

}




@NonCPS
def getNPMVersion(text) {
    def matcher = text =~ /"version"\s*:\s*"([^"]+)"/
    matcher ? matcher[0][1] : null
}

@NonCPS
def isRelease() {
    env.BRANCH_NAME == "master" || env.BRANCH_NAME ==~ /^maintenance\/.+/
}
