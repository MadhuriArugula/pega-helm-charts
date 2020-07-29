#!/usr/bin/env groovy
def bintrayautomation = "bintrayautomation"
def labels = ""
def bintrayPackageVersion = "1.0.0" 

node("pc-2xlarge") {

      stage("Init"){
          if (env.CHANGE_ID) {
            pullRequest.labels.each{
            echo "label: $it"
            validateProviderLabel(it)
            labels += "$it,"
            }
            labels = labels.substring(0,labels.length()-1)
            echo "PR labels -> $labels"
            // Comment on PR about the text execution
            pullRequest.comment("Starting ${env.BRANCH_NAME} validation on -> $labels")
            sh "curl -fsSL -o helm-v3.2.4-linux-amd64.tar.gz https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz"
            sh "tar -zxvf helm-v3.2.4-linux-amd64.tar.gz"
            sh "mv linux-amd64/helm /usr/local/bin/helm"
          } else {
            currentBuild.result = 'ABORTED'
            throw new Exception("Aborting as this is not a PR job")
         }
      }
      stage ("Checkout and Package Charts") {

            // Checkout PR Code
            def scmVars = checkout scm
            branchName = "${scmVars.GIT_BRANCH}"
            packageName = currentBuild.displayName
            prNumber = "${env.BRANCH_NAME}".split("-")[1]
            
            // Perform Chart packaging
            sh "helm dependency update ./charts/pega/"
            sh "helm dependency update ./charts/addons/"
            sh "curl -o index.yaml https://dl.bintray.com/pegasystems/helm-test-automation/index.yaml"
            sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/pega/"
            sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/addons/"
            sh "helm repo index --merge index.yaml --url https://dl.bintray.com/pegasystems/helm-test-automation/ ."
            sh "cat index.yaml"
            
            // Publish helm charts to test-automation repository
            withCredentials([usernamePassword(credentialsId: "bintrayautomation",
              passwordVariable: 'BINTRAY_APIKEY', usernameVariable: 'BINTRAY_USERNAME')]) {
                chartVersion = "${prNumber}.${env.BUILD_NUMBER}"
                pega_chartName = "pega-${chartVersion}.tgz"
                addons_chartName = "addons-${chartVersion}.tgz"
                DELETE_STATUS_CODE = sh(script: "curl -X DELETE -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/index.yaml --write-out '%{http_code}'", returnStdout: true).trim()
                PEGA_STATUS_CODE = sh(script: "curl -T --write-out '%{http_code}' ${pega_chartName} -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/", returnStdout: true).trim()
                ADDONS_STATUS_CODE = sh(script: "curl -T --write-out '%{http_code}' ${addons_chartName} -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/", returnStdout: true).trim()
                UPDATE_STATUS_CODE = sh(script: "curl -T --write-out '%{http_code}' index.yaml -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/", returnStdout: true).trim()
                PUBLISH_STATUS_CODE = sh(script: "curl -X POST -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/publish --write-out '%{http_code}'", returnStdout: true).trim()
                
                if ( "${DELETE_STATUS_CODE}" != 200 || "${PEGA_STATUS_CODE}" != 200 || "${ADDONS_STATUS_CODE}" != 200 
                      || "${UPDATE_STATUS_CODE}" != 200|| "${PUBLISH_STATUS_CODE}" != 200 ) {
                    currentBuild.result = 'FAILURE'
                    pullRequest.comment("Unable to publish helm charts to bintray repository. Please retry")
                }

            } 
      }

      stage("Setup Cluster and Execute Tests") {
          
          jobMap = [:]
          jobMap["job"] = "../kubernetes-test-orchestrator/US-366319"
          jobMap["parameters"] = [
                                  string(name: 'PROVIDERS', value: labels),
                                  string(name: 'HELM_CHART_VERSION', value: chartVersion),
                              ]
          jobMap["propagate"] = true
          jobMap["quietPeriod"] = 0 
          resultWrapper = build jobMap
          currentBuild.result = resultWrapper.result
          echo "Into Trigger Orchestrator"
      } 

  }

def validateProviderLabel(String provider){
    def validProviders = ["integ-all","integ-eks","integ-gke","integ-aks"]
    if(!validProviders.contains(provider)){
        currentBuild.result = 'FAILURE'
        pullRequest.comment("Invalid provider label - ${provider}. valid labels are ${validProviders}")
        throw new Exception("Invalid provider label - ${provider}. valid labels are ${validProviders}")
    }
}
