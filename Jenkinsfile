#!groovy

/**
 * This program and the accompanying materials are made available under the terms of the
 * Eclipse Public License v2.0 which accompanies this distribution, and is available at
 * https://www.eclipse.org/legal/epl-v20.html
 *
 * SPDX-License-Identifier: EPL-2.0
 *
 * Copyright IBM Corporation 2018, 2019
 */

node('ubuntu1804-docker-4c-4g') {
docker.image('markackertca/zowe-base').inside {
  def lib = library("jenkins-library").org.zowe.jenkins_shared_library

  // ============================== constants ==============================
  def GITHUB_REPOSITORY = 'zowe/docs-site'
  def isMasterBranch = env.BRANCH_NAME == 'master'
  def isReleaseBranch = env.BRANCH_NAME ==~ /^v[0-9]+\.[0-9]+\.[0-9x]+$/

  // ============================== variables ==============================
  // If run the publish step. Only take effect when building on "master" and 
  // "v?.?.x" branches. This option provides a chance to disable publish on
  // those branches.
  def RUN_PUBLISH = true
  // Target branch to publish. By default will be "gh-pages". If you specify a
  // branch name starts with "gh-pages-test", you can always perform test
  // publishing.
  def PUBLISH_BRANCH = 'gh-pages-test'
  // Target URL path to publish. Default is "stable" for master branch, "v?.?.x"
  // for v?.?.x release branches.
  def PUBLISH_PATH = ''
  // If run links check on all latest and archived versions, not only checking
  // current build.
  def FULL_SITE_LINKS_CHECK = false

  // below variables will be adjusted based on other variable values
  def allowPublishing = false
  def isTestPublishing = PUBLISH_BRANCH.startsWith('gh-pages-test')
  def publishTargetPath = 'stable'
  def githubDeploy = null

  // if we are on master, or v?.?.? / v?.?.x branch, we allow publish
  // if we publish target branch to test branch, we allow it anyway
  if (isMasterBranch || isReleaseBranch || isTestPublishing) {
    allowPublishing = true
  }
  if (allowPublishing && isReleaseBranch) {
    publishTargetPath = env.BRANCH_NAME
  }
  if (allowPublishing && PUBLISH_PATH) { // this is manually assigned parameter value
    publishTargetPath = PUBLISH_PATH
  }
  def publishTargetPathConverted = publishTargetPath.replaceAll(/\./, '-')

  def pipeline = lib.pipelines.nodejs.NodeJSPipeline.new(this)

  pipeline.admins.add("jackjia.ibm")

  pipeline.setup(
    packageName: 'org.zowe.docs',
    github: [
      email                      : lib.Constants.DEFAULT_GITHUB_ROBOT_EMAIL,
      usernamePasswordCredential : 'zowe-github',
    ],
    // FIXME: ideally this should set to false (using default by remove this line)
    ignoreAuditFailure            : true
  )

  pipeline.createStage(
    name          : "Prepare .deploy",
    isSkippable   : true,
    stage         : {
      // prepare .deploy folder
      // checkout PUBLISH_BRANCH to .deploy folder
      githubDeploy = lib.scm.GitHub.new(this)
      if (!githubDeploy) {
          error 'Failed to initialize GitHub instance.'
      }
      githubDeploy.init([
        repository                 : TEST_REPORSITORY,
        branch                     : PUBLISH_BRANCH,
        folder                     : '.deploy',
        email                      : lib.Constants.DEFAULT_GITHUB_ROBOT_EMAIL,
        usernamePasswordCredential : 'zowe-github',
      ])
      githubDeploy.cloneRepository()

      if (isMasterBranch) {
        // alway try to update default pages from master branch
        sh 'cp -r gh-pages-default/. .deploy/'
      }
    },
    timeout: [time: 20, unit: 'MINUTES']
  )

  // we have a custom build command
  pipeline.build(
    operation: {
      ansiColor('xterm') {
        sh 'npm install'
        sh "PUBLISH_TARGET_PATH=${publishTargetPath} " +
           "NODE_OPTIONS=--max_old_space_size=4096 " +
           "npm run docs:build"
      }
    }
  )

  pipeline.test(
    name          : 'Broken Links',
    operation         : {
      ansiColor('xterm') {
        // list all files generated
        sh "find .deploy | grep -v '.deploy/.git'"
        // check broken links
        timeout(30) {
          if (FULL_SITE_LINKS_CHECK) {
            sh 'npm run test:links'
          } else {
            sh "npm run test:links -- --start-point /${publishTargetPathConverted}/"
          }
        }
      }
    },
    allowMissingJunit : true
  )

  pipeline.createStage(
    name          : "Generate PDF",
    isSkippable   : true,
    stage         : {
      ansiColor('xterm') {
        timeout(time: 1, unit: 'HOURS') {
          sh 'npm run docs:pdf'
        }
        if (fileExists('.deploy/.pdf/out/Zowe_Documentation.pdf')) {
          sh "cp .deploy/.pdf/out/Zowe_Documentation.pdf .deploy/${publishTargetPathConverted}/"
        } else {
          error 'Failed to generate PDF document.'
        }
        // clean up pdf tmp folder
        echo 'Cleaning up .deploy/.pdf ...'
        sh 'rm -fr .deploy/.pdf || true'
      }
    },
    timeout: [time: 20, unit: 'MINUTES']
  )

  // define we need publish stage
  pipeline.publish(
    shouldExecute : {
      return allowPublishing && RUN_PUBLISH
    },
    operation: {
      githubDeploy.commit("deploy from ${env.JOB_NAME}#${env.BUILD_NUMBER}")
      githubDeploy.push()
    }
  )

  pipeline.end()
}
}
