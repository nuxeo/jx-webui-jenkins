/*
 * (C) Copyright 2019 Nuxeo (http://nuxeo.com/) and others.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 * Contributors:
 *     Antoine Taillefer <ataillefer@nuxeo.com>
 *     Nelson Silva <nsilva@nuxeo.com>
 */
 properties([
  [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/nuxeo/jx-webui-jenkins/'],
  [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', daysToKeepStr: '60', numToKeepStr: '60', artifactNumToKeepStr: '5']],
  disableConcurrentBuilds(),
])

void setGitHubBuildStatus(String context, String message, String state) {
  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/nuxeo/jx-webui-jenkins'],
    contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: context],
    statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'AnyBuildResult', message: message, state: state]]],
  ])
}

def VERSION

pipeline {
  agent {
    label 'jenkins-jx-base'
  }
  environment {
    ORG = 'nuxeo'
  }
  stages {
    stage('Build and push image') {
      steps {
        setGitHubBuildStatus('build', 'Build and push image', 'PENDING')
        container('jx-base') {
          script {
            VERSION =  sh(returnStdout: true, script: 'jx-release-version').trim()
            if (BRANCH_NAME != 'master') {
              VERSION += "-${BRANCH_NAME}";
            }
          }
          withEnv(["VERSION=${VERSION}"]) {
            echo "Building nuxeo/webui-jenkinsx:${VERSION}"
            sh """
              envsubst < skaffold.yaml > skaffold.yaml~gen
              skaffold build -f skaffold.yaml
            """
          }
        }
      }
      post {
        always {
          step([$class: 'JiraIssueUpdater', issueSelector: [$class: 'DefaultIssueSelector'], scm: scm])
        }
        success {
          setGitHubBuildStatus('build', 'Build and push image', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('build', 'Build and push image', 'FAILURE')
        }
      }
    }
    stage('Release') {
      when {
        branch 'master'
      }
      steps {
        setGitHubBuildStatus('release', 'Release', 'PENDING')
        container('jx-base') {
          withEnv(["VERSION=${VERSION}"]) {
            echo "Releasing ${VERSION}"
            sh """
              # ensure we're not on a detached head
              git checkout master

              # create the Git credentials
              jx step git credentials
              git config credential.helper store

              # Git tag
              jx step tag -v ${VERSION}

              # Git release
              jx step changelog -v v${VERSION}

              # update Jenkins image version in the platform env
              ./updatebot.sh ${VERSION}
            """
          }
        }
      }
      post {
        always {
          step([$class: 'JiraIssueUpdater', issueSelector: [$class: 'DefaultIssueSelector'], scm: scm])
        }
        success {
          setGitHubBuildStatus('release', 'Release', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('release', 'Release', 'FAILURE')
        }
      }
    }
  }
}
