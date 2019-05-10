import groovy.json.JsonSlurper

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
    sh "curl -S -H \"Content-Type: application/json\" -d '{\"content\":[{\"hash\": {\"BYTES_VALUE\" : \"${bytesValue}\"}}], \"address\": [{\"deployment\":\"${deploymentNameWoPath}\"}], \"operation\":\"add\", \"enabled\":\"true\"}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management > result2.txt"  
  }
}


@NonCPS
def determineFileName(path) {
  def idx = path.lastIndexOf('/')
  // Groovy method path.drop(idx) throws a ScriptSecurityException
  return path.substring(idx+1,path.length())
}
