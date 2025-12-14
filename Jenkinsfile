#!groovy​

def AGENT_NAME = 'maven'

properties([
  [$class: 'BuildDiscarderProperty',
   strategy: [$class: 'LogRotator', numToKeepStr: '10']]
])

stage('build') {
    node(AGENT_NAME) {
        checkout scm
        def v = version()
        currentBuild.displayName = "${env.BRANCH_NAME}-${v}-${env.BUILD_NUMBER}"
        mvn "clean verify"
    }
}

stage('build docker image') {
    node(AGENT_NAME) {
        mvn "clean package docker:build -DskipTests"
    }
}

def branch_type = get_branch_type(env.BRANCH_NAME)
def branch_deployment_environment = get_branch_deployment_environment(branch_type)

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

/* ---------- Utility functions ---------- */

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

def mvn(String goals) {
    def mvnHome  = tool "Maven-3.2.3"
    def javaHome = tool "JDK1.8.0_102"
    withEnv([
        "JAVA_HOME=${javaHome}",
        "PATH+MAVEN=${mvnHome}/bin"
    ]) {
        sh "mvn -B ${goals}"
    }
}

def version() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    return matcher ? matcher[0][1] : null
}
