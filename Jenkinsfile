#!/usr/bin/env groovy
def awsCredentialsId_PE = "CloudENG-Development"
def labels = ""
def pega_chartName = ""
def addons_chartName = ""
node("pc-2xlarge") {

  stage("Initialze"){
      if (env.CHANGE_ID) {
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
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/pega/"
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/addons/"
     withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
								credentialsId: awsCredentialsId_PE,
								accessKeyVariable: 'AWS_ACCESS_KEY_ID',
								secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                        ]) {
      pega_chartName = "pega-${prNumber}.${env.BUILD_NUMBER}.tgz"
      addons_chartName = "addons-${prNumber}.${env.BUILD_NUMBER}.tgz"
      sh "aws s3 cp ${pega_chartName} s3://kubernetes-pipeline/helm/"
      sh "aws s3 cp ${addons_chartName} s3://kubernetes-pipeline/helm/"
    }
  }

 stage("Trigger Orchestrator") {
    jobMap = [:]
    jobMap["job"] = "../kubernetes-test-orchestrator/US-366319"
    jobMap["parameters"] = [
                            string(name: 'PROVIDERS', value: labels),
                            string(name: 'HELM_CHART_NAME', value: pega_chartName+","+addons_chartName),
                        ]
    jobMap["propagate"] = true
    jobMap["quietPeriod"] = 0 
    resultWrapper = build jobMap
    currentBuild.result = resultWrapper.result
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