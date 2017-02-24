//TODO - Make SVN and GIT Checkout steps perfect with Jenkins way. Do not use Shell way.

def temporaryDockerRegistry = 'ec2-13-55-19-58.ap-southeast-2.compute.amazonaws.com'
def permanentDockerRegistry = 'ec2-13-54-206-33.ap-southeast-2.compute.amazonaws.com'
node {
  //--------------------------------------
  stage('Code Pickup') {
    echo "Source Code Repository Type : ${scmSourceRepo}"
    echo "Source Code Repository Path : ${scmPath}"
    
    if("${scmSourceRepo}".toUpperCase()=='SVN'){
        //Not a perfect solution. Reimplement with SVN Step or checkout Step
        sh "svn co --username ${scmUsername} --password ${scmPassword} ${scmPath} ."
        
    } else if("${scmSourceRepo}".toUpperCase()=='GIT'){
        //Not a perfect Solution. Reimplement with Git Step or checkout Step
        scmPath = scmPath.substring(0, scmPath.indexOf("//")+2) + scmUsername + ":" + scmPassword + "@" +scmPath.substring(scmPath.indexOf("//")+2, scmPath.length());
        //echo "GIT PATH: ${scmPath}"
        try {
            //If we use git clone, it will not clone in the same path if we rebuild the pipeline
            sh 'ls -a | xargs rm -fr' 
        } catch (error) {
        }
        sh "git clone ${scmPath} ."
        
        //The below solutions not working with username and password
        //git "${scmPath}" //Alternate option
        //checkout scm: [$class:'GitSCM', userRemoteConfigs: [[url: scmPath ]]]
    } else {
        error 'Unknown Source code repository. Only GIT and SVN are supported'
    }
  } 
  //---------------------------------------
  
  //Preparing for Build & Package
  def appModuleSeperated = fileExists 'app'
  def testModuleSeperated = fileExists 'test'
  def appPath = ''
  def testPath = ''  
  if (appModuleSeperated) {
    echo 'There is a directory called app and hence assuming that /app is the working directory for application.'
    appPath='app/'
  } else {
    echo 'There is no directory named app. Hence assuming that the current directory is the working directory.'
    appPath = ''
  }

  if (testModuleSeperated) {
    echo 'There is a directory called test and hence assuming that /test is the integration test automation suite.'
    testPath = 'test/'  
  } else {
    echo 'There is no directory named test. Hence assuming that there is no integration testing automation suite.'
    testPath = ''  
  }
  
  def isBuildAndPackageRequired = true
  def buildDockerFile = appPath + 'Dockerfile.build'
  def distDockerFile = appPath + 'Dockerfile.dist'
  if (fileExists(buildDockerFile) && fileExists(distDockerFile)) {
    echo 'It looks like this application is compiler based application and hence there is a seperate dockerfile found for compile, build and packaging.'
    isBuildAndPackageRequired = true;    
  } else if (appPath + fileExists('Dockerfile')) {
    echo 'It looks like there is only one docker file. May be this is interpreter based application technology.'
    isBuildAndPackageRequired = false;
    distDockerFile = appPath + 'Dockerfile'
  } else {
    echo 'Dockerfile not found under ' + appPath
  }
  def appWorkingDir = (appPath=='') ? '.' : appPath.substring(0, appPath.length()-1)
  //End of Preparation for Build and Package
  
  //TODO - Tune it later. Dirty solution to identify the Jenkins generated artifacts for Nexus
        sh 'echo Nexus>Nexus.txt'
  //Dirty solution ends.
  
  //---------------------------------------
  if(isBuildAndPackageRequired){
    echo 'Compile and Package runs as seperate stage due to this app requirements.'
    stage('Compile, Unit Test & Package') {
      echo 'Working Directory for Docker Build file: ' + appWorkingDir
      echo "Build Tag Name: ${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}"
      echo "Build params: --file ${buildDockerFile} ${appWorkingDir}"
      
      appCompileAndPackageImg = docker.build("${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}", "--file ${buildDockerFile} ${appWorkingDir}")      
      
      //Reading the CMD from Docker file and would be executing within the container. This is due to the behaviour of this plugin
      //TODO - Danger zone. This approach is not based on grammer. So in case if the CMD is not shell (if exec) or ENTRYPOINT is given, this approach would not work
      def dockerCMD = readFile buildDockerFile
      echo dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())      
      appCompileAndPackageImg.inside('--net=host') {        
        sh dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())
      }
      //End of Danger Zone code      
    }
  } else {
    echo 'There is no Compile and Package as seperate step'
  }
  //---------------------------------------
  
  if("${stage}".toUpperCase() == 'BUILD') {
    echo 'The requested stage is Build only. Hence pushing the successful image into temporary repo'
    //---------------------------------------
    // We are pushing to a private Temporary Docker registry as this is just Build case.
    // 'docker-registry-login' is the username/password credentials ID as defined in Jenkins Credentials.
    // This is used to authenticate the Docker client to the registry.
    //docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
    //withDockerRegistry([credentialsId: 'docker-registry-login', url: temporaryDockerRegistry]) {
   docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Stage') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push();
      }
    }   
  } else if ("${stage}".toUpperCase() == 'DEPLOY') {
    echo 'The requested stage is Deploy. Hence without certifying, the image is pushed to permanent repo'
    //withDockerRegistry([credentialsId: 'docker-registry-login', url: permanentDockerRegistry]) {
    docker.withRegistry("https://${permanentDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Publish') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push('latest');
      }
    }    
  } else if ("${stage}".toUpperCase() == 'CERTIFY'){
    echo 'The requested stage is Certify. Hence just publishing to temporary repo and provisioning sandbox'
    docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Certify') {
        // Let us tag and push the newly built image. Will tag using the image name provided. No need of Docker hostname as it appends itself.
        //pcImg = docker.build("${temporaryDockerRegistry}/${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push('SNAPSHOT');
      }
    }   
  }
  //---------------------------------------
  
  stage('Publish Jenkins Output to Nexus'){
    //TODO in code - Tune it later. Dirty solution to identify the Jenkins generated artifacts for Nexus. 
    //TODO in Jenkins - Needs a Credential with the name "Nexus"
    //TODO in Nexus web - Created a Hosted Site Repository with the name MEC
    //TODO - Nexus3 support - Check the Sonatype plugin once released
    //Nexus is a great component artifact repo. Does not look great dealing with binary documents and intermediate outputs.
    //The plugin is too weak. It can upload only one file and hence Zipped
    echo 'Publishing the artifacts...';
    //def PWD = pwd(); //"${PWD}/artifacts.tar.gz"
    sh 'find . -type f -newer Nexus.txt -print0 | tar -czvf artifacts.tar.gz --ignore-failed-read --null -T -'
    //Nexus 2
    //nexusArtifactUploader artifacts: [[artifactId: "${env.JOB_NAME}", classifier: '', file: 'artifacts.tar.gz', type: 'gzip']], credentialsId: 'Nexus', groupId: 'org.jenkins-ci.main.mec', nexusUrl: '13.55.146.108:8085/nexus', nexusVersion: 'nexus2', protocol: 'http', repository: 'MEC',version: "${env.BUILD_NUMBER}"
    //Nexus 3
    nexusArtifactUploader artifacts: [[artifactId: "${env.JOB_NAME}", classifier: '', file: 'artifacts.tar.gz', type: 'gzip']], credentialsId: 'Nexus', groupId: 'org.jenkins-ci.main.mec', nexusUrl: '13.55.146.108:8084', nexusVersion: 'nexus3', protocol: 'http', repository: 'MEC',version: "${env.BUILD_NUMBER}"
    sh 'rm Nexus.txt'    
    //Dirty solution ends
  }  
}
