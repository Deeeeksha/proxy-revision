def String url = "${params.GIT_URL}"
def String[] res = url.split('/')
def String repoName = res[res.length - 1]
if (repoName.endsWith('.git')) repoName = repoName.substring(0, repoName.length() - 4)

def proxyName(jsonDir, jsonFile) {
  env.PROXYNAME = sh([script: "cat ${jsonFile.path} | grep 'title' | cut -d ':' -f 2 | sed 's/[[:space:]]//g'", returnStdout: true]).trim()
}

properties([
    parameters([
        [
            $class: 'ChoiceParameter',
            choiceType: 'PT_CHECKBOX',
            description: 'Check the Env Name from the List for the Deployment',
            // filterLength: 1, 
            // filterable: true,  
            name: 'ENVIRONMENT',
            script: [
                $class: 'GroovyScript',
                script: [ classpath: [], sandbox: true, script: 
                    """
                    return ['dev-1', 'qa', 'test']
                    """
                ]
            ]
        ]
    ])
])

pipeline {
  agent any
  environment {
        GCLOUD_DIR = "$JENKINS_HOME/google-cloud-sdk/bin"
        VACUUM_DIR = '/usr/local/bin/vacuum'
        APIGEE_CLI_DIR = "$HOME/.apigeecli/bin"
  }
  parameters {
        string(name: 'GIT_URL', defaultValue: '', description: 'Enter GitHub URL', trim: true)
        string(name: 'GIT_BRANCH', defaultValue: '', description: 'Enter GitHub Branch', trim: true)
        string(name: 'APIGEE_ORG', defaultValue: '', description: 'Enter the Apigee Organization', trim: true)
        string(name: 'APIGEE_REVISION', defaultValue: '', description: 'Enter the Apigee Revision', trim: true)
        booleanParam(name: 'SKIP_PROXY_CREATION', defaultValue: true, description: 'Check the box if the proxy is already created and you only want to deploy the proxy!')
        booleanParam(name: 'SKIP_PROXY_DEPLOYMENT', defaultValue: true, description: 'Check the box if you only want to create the proxy and do not want it to be deployed!')
  }
    stages {
    // Install Dependencies
    stage('Installing Dependencies') {
      steps {
          sh '''#!/bin/bash
                echo "Checking for pre-installed dependencies..."
                echo ""
                if [ ! -d "$GCLOUD_DIR" ]; then
                    echo "Installing GCloud CLI..."
                    echo ""
                    cd $JENKINS_HOME
                    curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-412.0.0-linux-x86_64.tar.gz
                    tar -xf google-cloud-cli-412.0.0-linux-*.tar.gz
                    ./google-cloud-sdk/install.sh -q
                    source $JENKINS_HOME/google-cloud-sdk/completion.bash.inc
                    source $JENKINS_HOME/google-cloud-sdk/path.bash.inc
                else
                    echo "GCloud CLI is already Installed!"
                    echo ""
                fi

                if [ ! -f "$VACUUM_DIR" ]; then
                    echo "Installing Vacuum Lint..."
                    echo ""
                    curl -fsSL https://quobix.com/scripts/install_vacuum.sh | sh
                else
                    echo "Vacuum Lint is already Installed!"
                    echo ""
                fi

                if [ ! -d "$APIGEE_CLI_DIR" ]; then
                    echo "Installing Apigee CLI..."
                    echo ""
                    curl -L https://raw.githubusercontent.com/apigee/apigeecli/main/downloadLatest.sh | sh -
                else
                    echo "Apigee CLI is already Installed!"
                    echo ""
                fi
             '''
      }
    }
    // Logging into GCloud
    stage('Logging into Google Cloud and Get Access Token') {
      steps {
        script {
          withCredentials([file(credentialsId: 'GAccountKey', variable: 'GOOGLE_SERVICE_ACCOUNT_KEY')]) {
          sh '${GCLOUD_DIR}/gcloud auth activate-service-account --key-file ${GOOGLE_SERVICE_ACCOUNT_KEY}'
          env.TOKEN = sh([script: "${GCLOUD_DIR}/gcloud auth print-access-token", returnStdout: true ]).trim()
          }
        }
      }
    }
    // Cloning Git
    stage('Cloning Git') {
      steps {
        script {
            checkout([$class: 'GitSCM', branches: [[name: "*/${params.GIT_BRANCH}"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: repoName]], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'GitHubCreds', url: url]]])
        }
      }
    }
    // Check Linting For The Proxy
    stage('Checking Linting for the Proxy') {
      steps {
        script {
          def jsonDir = "${repoName}"
          def jsonFiles = findFiles glob: "${jsonDir}/*.json"
          for (jsonFile in jsonFiles) {
            sh "vacuum lint -d ${jsonFile.path} -r url-formatter.yaml"
          }
        }
      }
    }
    // Creating Proxy to Apigee
    stage('Creating Proxy') {
      steps {
        script {
            if (params.SKIP_PROXY_CREATION == false) {
            def jsonDir = "${repoName}"
            def jsonFiles = findFiles glob: "${jsonDir}/*.json"
            for (jsonFile in jsonFiles) {
                proxyName(jsonDir, jsonFile)
                sh "$APIGEE_CLI_DIR/apigeecli apis create openapi -f ${jsonFile.path} -o ${APIGEE_ORG} -n ${env.PROXYNAME} -t ${env.TOKEN}"
            } 
          }
        }
      }
    }
    // Deploying Proxy to Apigee
    stage('Deploying Proxy') {
      parallel {
        stage('DEV') {
        when { 
          allOf {
            expression { return ENVIRONMENT.contains('dev-1') }
            expression { params.SKIP_PROXY_DEPLOYMENT == false }
          }
        }
        steps {
          script{
            def jsonDir = "${repoName}"
            def jsonFiles = findFiles glob: "${jsonDir}/*.json"
            for (jsonFile in jsonFiles) {
                proxyName(jsonDir, jsonFile)
                sh "$APIGEE_CLI_DIR/apigeecli apis deploy -o ${APIGEE_ORG} -n ${env.PROXYNAME} -e dev-1 -v ${APIGEE_REVISION}  -t ${env.TOKEN} --wait"
              }
            }
          }
        }
        stage('QA') {
        when { 
          allOf {
            expression { return ENVIRONMENT.contains('qa') }
            expression { params.SKIP_PROXY_DEPLOYMENT == false }
          }
        }
        steps {
          script {
            def jsonDir = "${repoName}"
            def jsonFiles = findFiles glob: "${jsonDir}/*.json"
            for (jsonFile in jsonFiles) {
                proxyName(jsonDir, jsonFile)
                sh "$APIGEE_CLI_DIR/apigeecli apis deploy -o ${APIGEE_ORG} -n ${env.PROXYNAME} -e qa -v ${APIGEE_REVISION}  -t ${env.TOKEN} --wait"
              }
            }
          }
        }
        stage('TEST') {
        when {
          allOf {
            expression { return ENVIRONMENT.contains('test') }
            expression { params.SKIP_PROXY_DEPLOYMENT == false }
          }
        }
        steps {
          script {
            def jsonDir = "${repoName}"
            def jsonFiles = findFiles glob: "${jsonDir}/*.json"
            for (jsonFile in jsonFiles) {
                proxyName(jsonDir, jsonFile)
                sh "$APIGEE_CLI_DIR/apigeecli apis deploy -o ${APIGEE_ORG} -n ${env.PROXYNAME} -e test -v ${APIGEE_REVISION}  -t ${env.TOKEN} --wait"
              }
            }
          }
        }
      }
    }
  }
post('Post-build steps') {
  always {
    script {
      currentBuild.result = currentBuild.result == null ? "SUCCESS" : currentBuild.result
      }
      logstashSend failBuild: true, maxLines: 1000
    }
  }
}
