import groovy.json.JsonSlurper

node {
  stage 'Commit Stage'

    git branch: '10.x', url: 'https://github.com/wildfly/quickstart.git'
    env.JAVA_HOME="${tool 'JDK_8'}"
    env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"
    def mvnHome = tool name: 'maven_3.3.3', type: 'hudson.tasks.Maven$MavenInstallation'
    sh "cd kitchensink-angularjs/ && ${mvnHome}/bin/mvn clean install"
  
  stage 'Deploy Stage'
    def warFiles = findFiles glob: '**/target/*.war'
    for (int i=0; i<warFiles.size(); i++) {
    deploy(warFiles[i].path)
    echo "-----------------------------TESTST------------------------------------"  
    echo warFiles[i].path  
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
      sh "curl -S -H \"Content-Type: application/json\" -d '{\"content\":[{\"hash\": {\"BYTES_VALUE\" : \"${bytesValue}\"}}], \"address\": [{\"deployment\":\"${deploymentNameWoPath}\"}], \"operation\":\"add\", \"enabled\":\"true\"}' --digest http://${env.wildflyMgmtUser}:${env.wildflyMgmtPassword}@${hostname}:${managementPort}/management > result2.txt"
      def deployResult = readFile 'result2.txt'
      def failure = hasFailure(deployResult)
      if (failure != null) {
        error "Deployment of ${deploymentNameWoPath} failed with error: ${failure}"
      }
    } else {
      // fail build as deployment was not successfull
      error "Upload of ${deploymentFileName} failed"
    }
  }
}

@NonCPS
def hasFailure(result) {
  def jsonSlurper = new JsonSlurper()
  def object = jsonSlurper.parseText(result)
  if (object.outcome != 'success') {
    return object.'failure-description'
  } else {
    return null
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
