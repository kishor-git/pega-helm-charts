#!/usr/bin/env groovy
def awsCredentialsId_PE = "CloudENG-Development"
def labels = ""
def pega_chartName = ""
def addons_chartName = ""
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
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/pega/"
    sh "helm package --version ${prNumber}.${env.BUILD_NUMBER} ./charts/addons/"
     withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
								credentialsId: awsCredentialsId_PE,
								accessKeyVariable: 'AWS_ACCESS_KEY_ID',
								secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
      withCredentials([usernamePassword(credentialsId: "bin.pega.io",
                passwordVariable: 'ARTIFACTORY_PASSWORD', 
                usernameVariable: 'ARTIFACTORY_USER')]) {
      pega_chartName = "pega-${prNumber}.${env.BUILD_NUMBER}.tgz"
      addons_chartName = "addons-${prNumber}.${env.BUILD_NUMBER}.tgz"
      sh "aws s3 cp ${pega_chartName} s3://kubernetes-pipeline/helm/"
      sh "aws s3 cp ${addons_chartName} s3://kubernetes-pipeline/helm/"
      sh "export MD5SUM=\$(md5sum ${pega_chartName} | awk '{print \$1}')"
      sh "export SHA1SUM=\$(sha1sum ${pega_chartName} | awk '{print \$1}')"
      sh "export SHA256SUM=\$(sha256sum ${pega_chartName} | awk '{print \$1}')"
      sh 'curl -XPUT --user ${ARTIFACTORY_USER}:${ARTIFACTORY_PASSWORD} \
               --upload-file ${pega_chartName} -H"X-Checksum-Sha256:${SHA256SUM}" -H"X-Checksum-Sha1:${SHA1SUM}" -H"X-Checksum-Md5:${MD5SUM}" \
               https://bin.pega.io/artifactory/helm-local/${pega_chartName}'
      // sh "helm repo add pega-artifactory https://bin.pega.io/artifactory/helm-stable/ --username=${ARTIFACTORY_USER} --password=${ARTIFACTORY_PASSWORD}"
      // sh "helm search repo pega-artifactory"
      // sh "curl -fsSL -o jfrog https://api.bintray.com/content/jfrog/jfrog-cli-go/1.38.0/jfrog-cli-linux-386/jfrog?bt_package=jfrog-cli-linux-386"
      // sh "chmod 777 jfrog"
      // sh "export CI=true"
      // sh "./jfrog rt u pega-${prNumber}.${env.BUILD_NUMBER}.tgz helm-local --url=https://bin.pega.io/artifactory/helm-local/ --user=${ARTIFACTORY_USER} --password=${ARTIFACTORY_PASSWORD}"
    }
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
