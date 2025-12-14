#!groovy

def AGENT_NAME = 'maven'

/* ===================== PROPERTIES ===================== */
properties([
  [$class: 'BuildDiscarderProperty',
   strategy: [$class: 'LogRotator', numToKeepStr: '10']]
])

/* ===================== MAVEN INSTALL ===================== */
def ensureMaven(String ver = "3.2.3") {
    def dir = "${env.WORKSPACE}/.tools/apache-maven-${ver}"
    def tgz = "apache-maven-${ver}-bin.tar.gz"
    def url = "https://archive.apache.org/dist/maven/maven-3/${ver}/binaries/${tgz}"

    sh """
      set -e
      mkdir -p "${env.WORKSPACE}/.tools"
      if [ ! -d "${dir}" ]; then
        echo "Installing Maven ${ver}"
        cd "${env.WORKSPACE}/.tools"
        rm -f "${tgz}"
        (command -v curl >/dev/null 2>&1 && curl -fsSL -o "${tgz}" "${url}") || \
        (command -v wget >/dev/null 2>&1 && wget -q -O "${tgz}" "${url}")
        tar -xzf "${tgz}"
      fi
      "${dir}/bin/mvn" -v
    """
    return dir
}

def mvn(String goals) {
    def mvnHome = ensureMaven("3.2.3")
    sh "${mvnHome}/bin/mvn -B ${goals}"
}

/* ===================== BUILD ===================== */
stage('build') {
    node(AGENT_NAME) {
        checkout scm
        def v = version()
        currentBuild.displayName = "${env.BRANCH_NAME}-${v}-${env.BUILD_NUMBER}"
        mvn "clean verify"
    }
}

/* ===================== DOCKER BUILD ===================== */
stage('build docker image') {
    node(AGENT_NAME) {
        mvn "clean package docker:build -DskipTests"
    }
}

/* ===================== BRANCH LOGIC ===================== */
def branch_type = get_branch_type(env.BRANCH_NAME)
def branch_deployment_environment = get_branch_deployment_environment(branch_type)

/* ===================== DEPLOY ===================== */
if (branch_deployment_environment) {
    stage('deploy') {
        if (branch_deployment_environment == "prod") {
            timeout(time: 1, unit: 'DAYS') {
                input "Deploy to ${branch_deployment_environment}?"
            }
        }
        node(AGENT_NAME) {
            sh "echo Deploying to ${branch_deployment_environment}"
        }
    }

    if (branch_deployment_environment != "prod") {
        stage('integration tests') {
            node(AGENT_NAME) {
                sh "echo Running integration tests in ${branch_deployment_environment}"
            }
        }
    }
}

/* ===================== GITFLOW ===================== */
if (branch_type == "dev") {
    stage('start release') {
        timeout(time: 1, unit: 'HOURS') {
            input "Start a release?"
        }
        node(AGENT_NAME) {
            sshagent(['f1ad0f5d-df0d-441a-bea0-fd2c34801427']) {
                mvn("jgitflow:release-start")
            }
        }
    }
}

if (branch_type == "release") {
    stage('finish release') {
        timeout(time: 1, unit: 'HOURS') {
            input "Finish the release?"
        }
        node(AGENT_NAME) {
            sshagent(['f1ad0f5d-df0d-441a-bea0-fd2c34801427']) {
                mvn("jgitflow:release-finish -Dmaven.javadoc.skip=true -DnoDeploy=true")
            }
        }
    }
}

if (branch_type == "hotfix") {
    stage('finish hotfix') {
        timeout(time: 1, unit: 'HOURS') {
            input "Finish the hotfix?"
        }
        node(AGENT_NAME) {
            sshagent(['f1ad0f5d-df0d-441a-bea0-fd2c34801427']) {
                mvn("jgitflow:hotfix-finish -Dmaven.javadoc.skip=true -DnoDeploy=true")
            }
        }
    }
}

/* ===================== UTILITIES ===================== */
def get_branch_type(String branch_name) {
    if (branch_name =~ ".*development") return "dev"
    if (branch_name =~ ".*release/.*")  return "release"
    if (branch_name =~ ".*master")      return "master"
    if (branch_name =~ ".*feature/.*")  return "feature"
    if (branch_name =~ ".*hotfix/.*")   return "hotfix"
    return null
}

def get_branch_deployment_environment(String branch_type) {
    if (branch_type == "dev")     return "dev"
    if (branch_type == "release") return "staging"
    if (branch_type == "master")  return "prod"
    return null
}

def version() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    return matcher ? matcher[0][1] : null
}
