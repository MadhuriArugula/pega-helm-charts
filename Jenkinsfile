#!/usr/bin/env groovy
def bintrayautomation = "bintrayautomation"
def labels = ""
def bintrayPackageVersion = "1.0.0"
node("pc-2xlarge") {

  stage("Initialze"){
      if (env.CHANGE_ID) {
        //Just a comment
        pullRequest.comment("Starting pipeline for PR validation -> ${env.BRANCH_NAME}")
        pullRequest.labels.each{
        echo "label: $it"
        validateProviderLabel(it)
        labels += "$it,"
      }
        labels = labels.substring(0,labels.length()-1)
        echo "PR labels -> $labels"
     }else {
       currentBuild.result = 'ABORTED'
       throw new Exception("Aborting as this is not a PR job")
     }
 }
 stage ("Checkout and Package Charts") {

    sh "curl -fsSL -o helm-v3.2.4-linux-amd64.tar.gz https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz"
    sh "tar -zxvf helm-v3.2.4-linux-amd64.tar.gz"
    sh "mv linux-amd64/helm /usr/local/bin/helm"
    def scmVars = checkout scm
    branchName = "${scmVars.GIT_BRANCH}"
    packageName = currentBuild.displayName
    prNumber = "${env.BRANCH_NAME}".split("-")[1]
    sh "helm dependency update ./charts/pega/"
    sh "helm dependency update ./charts/addons/"
    sh "curl -o index.yaml https://dl.bintray.com/pegasystems/helm-test-automation/index.yaml"
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/pega/"
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/addons/"
    sh "helm repo index --merge index.yaml --url https://dl.bintray.com/pegasystems/helm-test-automation/ ."
    sh "cat index.yaml"
     withCredentials([usernamePassword(credentialsId: "bintrayautomation",
      passwordVariable: 'BINTRAY_APIKEY', usernameVariable: 'BINTRAY_USERNAME')]) {
      chartVersion = "${prNumber}.${env.BUILD_NUMBER}"
      pega_chartName = "pega-${chartVersion}.tgz"
      addons_chartName = "addons-${chartVersion}.tgz"
      sh "curl -X DELETE -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/index.yaml"
      sh "curl -T ${pega_chartName} -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/"
      sh "curl -T ${addons_chartName} -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/"
      sh "curl -T index.yaml -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/"
      sh "curl -X POST -u${BINTRAY_USERNAME}:${BINTRAY_APIKEY} https://api.bintray.com/content/pegasystems/helm-test-automation/helm-test-automation/${bintrayPackageVersion}/publish"
   } 
  }

 stage("Trigger Orchestrator") {
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
