import groovy.json.JsonSlurper

String gitUrl = "https://github.com/wildfly/quickstart.git"

String[] hostnames = ["localhost","localhost2"]

String deployBranch = '10.x'

String appDir = "kitchensink-angularjs/"

node {
  stage 'Checkout Stage'
    checkout([$class: 'GitSCM', branches: [[name: "$deployBranch" ]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git', url: "$gitUrl"]]]) 
  
  stage 'Commit Stage'
    dir("$appDir") {
      sh "/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/maven_3.3.3/bin/mvn clean install -DskipTests "
    }
  for(int j=0; j<hostnames.size();j++) {
      stage 'Deploy Stage ' + hostnames[j]
        def warFiles = findFiles glob: "$appDir/target/*.war"
        for (int i=0; i<warFiles.size(); i++) {
          deploy(warFiles[i].path, hostnames[j])
        }
  }    
  
}

def deploy(deploymentFileName,hostname) {
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'wildFlyManagementCredentials', passwordVariable: 'wildflyMgmtPassword', usernameVariable: 'wildflyMgmtUser']]) {
    
    def deploymentNameWoPath = determineFileName(deploymentFileName)
    def managementPort = '9990'
    // undeploy old war if present
    sh """curl -S -H "content-Type: application/json" -d '{"operation":"undeploy", "address":[{"deployment":"${deploymentNameWoPath}"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"""

    sh """curl -S -H "content-Type: application/json" -d '{"operation":"remove", "address":[{"deployment":"${deploymentNameWoPath}"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"""
    // step 1: upload archive
    sh """curl -F "file=@${deploymentFileName}" --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management/add-content > result.txt"""

node {
  stage 'Checkout Stage'
    checkout([$class: 'GitSCM', branches: [[name: '10.x']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git', url: "https://github.com/wildfly/quickstart.git"]]]) 
  
  stage 'Commit Stage'
    sh " cd kitchensink-angularjs/ && /var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/maven_3.3.3/bin/mvn clean install -DskipTests "
    
       
  stage 'Deploy Stage'
    def warFiles = findFiles glob: 'kitchensink-angularjs/target/*.war'
    for (int i=0; i<warFiles.size(); i++) {
    deploy(warFiles[i].path)
    }
}

def deploy(deploymentFileName) {
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'wildFlyManagementCredentials', passwordVariable: 'wildflyMgmtPassword', usernameVariable: 'wildflyMgmtUser']]) {
    def hostname = 'localhost'
    def managementPort = '9990'
    def deploymentNameWoPath = determineFileName(deploymentFileName)
    
    // undeploy old war if present
    sh "curl -S -H \"content-Type: application/json\" -d '{\"operation\":\"undeploy\", \"address\":[{\"deployment\":\"${deploymentNameWoPath}\"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"
    sh "curl -S -H \"content-Type: application/json\" -d '{\"operation\":\"remove\", \"address\":[{\"deployment\":\"${deploymentNameWoPath}\"}]}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management"
    // step 1: upload archive
    sh "curl -F \"file=@${deploymentFileName}\" --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management/add-content > result.txt"

    // step 2: deploy the archive
    // read result from step 1
    def uploadResult = readFile 'result.txt'
    def bytesValue = extractByteValue(uploadResult)
    if (bytesValue != null) {

      sh """curl -S -H "Content-Type: application/json" -d '{"content":[{"hash": {"BYTES_VALUE" : "${bytesValue}"}}], "address": [{"deployment":"${deploymentNameWoPath}"}], "operation":"add", "enabled":"true"}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management > result2.txt"""
      
      sh "curl -S -H \"Content-Type: application/json\" -d '{\"content\":[{\"hash\": {\"BYTES_VALUE\" : \"${bytesValue}\"}}], \"address\": [{\"deployment\":\"${deploymentNameWoPath}\"}], \"operation\":\"add\", \"enabled\":\"true\"}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management > result2.txt"
    } else {
      // fail build as deployment was not successfull
      error "Upload of ${deploymentFileName} failed"
    }
  }
}

@NonCPS
def extractByteValue(uploadResult) {
  // parse JSON
  def jsonSlurper = new JsonSlurper()
  def object = jsonSlurper.parseText(uploadResult)
  def result = null

  // check that upload was successfull
  if (object.outcome == 'success') {
    result = object.result.BYTES_VALUE
  }
  return result
}

@NonCPS
def determineFileName(path) {
  def idx = path.lastIndexOf('/')
  // Groovy method path.drop(idx) throws a ScriptSecurityException
  return path.substring(idx+1,path.length())
}
