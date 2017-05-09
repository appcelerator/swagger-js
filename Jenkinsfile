#! groovy
library 'pipeline-library'

timestamps {
  node('(osx || linux)') {
    def packageVersion = ''
    def isPR = false

    stage('Checkout') {
      checkout scm

      isPR = env.BRANCH_NAME.startsWith('PR-')
      packageVersion = jsonParse(readFile('package.json'))['version']
      currentBuild.displayName = "#${packageVersion}-${currentBuild.number}"
    }

    nodejs(nodeJSInstallationName: 'node 6.9.5') {
      ansiColor('xterm') {
        timeout(15) {
          stage('Build') {
            // Install yarn if not installed
            if (sh(returnStatus: true, script: 'which yarn') != 0) {
              sh 'npm install -g yarn'
            }
            sh 'yarn install'
            try {
              withEnv(['JUNIT_REPORT_PATH=junit_report.xml']) {
                sh 'yarn test'
              }
            } catch (e) {
              throw e
            } finally {
              junit 'junit_report.xml'
            }
            fingerprint 'package.json'
            // Don't tag PRs
            if (!isPR) {
              pushGitTag(name: packageVersion, message: "See ${env.BUILD_URL} for more information.", force: true)
            }
          } // stage
        } // timeout

        stage('Security') {
          // Clean up and install only production dependencies
          sh 'yarn install --production'

          // Scan for NSP and RetireJS warnings
          sh 'yarn global add nsp'
          sh 'nsp check --output summary --warn-only'

          sh 'yarn global add retire'
          sh 'retire --exitwith 0'

          step([$class: 'WarningsPublisher', canComputeNew: false, canResolveRelativePaths: false, consoleParsers: [[parserName: 'Node Security Project Vulnerabilities'], [parserName: 'RetireJS']], defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: ''])
        } // stage

        stage('Publish') {
          if (!isPR) {
            // sh 'npm publish'
            // Trigger client-generator job
            build job: "../client-generator/${env.BRANCH_NAME}", wait: false
          }
        } // stage
      } // ansiColor
    } //nodejs
  } // node
} // timestamps
