def APP_NAME = 'gcloud-pubsub-emulator'


pipeline {
  agent {
    label "jenkins-python-fbtest"
  }
  options {
    timeout(time: 15, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '30', daysToKeepStr: '30', artifactNumToKeepStr: '30', artifactDaysToKeepStr: '30'))
  }
  environment {
    ARTIFACTORY_CREDENTIALS = credentials('freshbooks-bot-artifactory')
    SLACK_TOKEN = credentials('slack-bot-token')
  }
  stages {
    stage('Build and publish') {
      steps {
        container('python') {
          sh "docker build --no-cache -t gcr.io/freshbooks-builds/${APP_NAME}:${env.BRANCH_NAME} ."
        }
      }
      post {
        success {
          container('python') {
            script {
              sh "docker push gcr.io/freshbooks-builds/${APP_NAME}:${env.BRANCH_NAME}"
              if (env.BRANCH_NAME == 'master') {
                sh "docker tag gcr.io/freshbooks-builds/${APP_NAME}:${env.BRANCH_NAME} gcr.io/freshbooks-builds/${APP_NAME}:latest"
                sh "docker push gcr.io/freshbooks-builds/${APP_NAME}:latest"
              }
            }
          }
        }
      }
    }
  }
  post {
    always {
      container('python') {
        sh "sudo pip install -i https://${ARTIFACTORY_CREDENTIALS_USR}:${ARTIFACTORY_CREDENTIALS_PSW}@freshbooks.jfrog.io/freshbooks/api/pypi/pypi/simple -r requirements-jenkins.txt"
        notifyStatus(currentBuild, SLACK_TOKEN)
      }
    }
  }
}

def notifyStatus(build, slackToken) {
  commits = fetchGitCommits(build)
  status = build.currentResult
  sh "freshbuilds slack -st ${slackToken} notify_status -s ${status} -r . -c ${commits}"
}

def fetchGitCommits(build) {
  def commits = []
  if (!(env.BRANCH_NAME ==~ /^PR-\d+$/)) {
    commits = getBuildChangeSets(build)
  }

  return commits.size() == 0 ? commitHashForBuild(build) : commits.join(',')
}

@NonCPS
def getBuildChangeSets(build) {
  def commits = []
  if (build == null) {
    return commits
  }
  if (build.rawBuild.changeSets.size() == 0) {
    commits.addAll(getBuildChangeSets(build.getPreviousBuild()))
  } else {
    build.rawBuild.changeSets.each {
      it.each {
        commits.add(it.commitId)
      }
    }
  }

  return commits
}

@NonCPS
def commitHashForBuild(build) {
  def scmAction = build?.rawBuild?.actions.find { 
    action -> action instanceof jenkins.scm.api.SCMRevisionAction 
  }

  if (scmAction?.revision == null) {
    return "${GIT_COMMIT}"
  }
  else if (scmAction?.revision instanceof org.jenkinsci.plugins.github_branch_source.PullRequestSCMRevision) {
    return scmAction?.revision.getPullHash()
  }

  return scmAction?.revision.getHash()
}
