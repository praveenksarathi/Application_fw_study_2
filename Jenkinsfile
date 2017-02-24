// This is a sample program to study Continous Integration 
def TempDocker = 'ec2-13-55-19-58.ap-southeast-2.compute.amazonaws.com'
def permDocker = 'ec2-13-54-206-33.ap-southeast-2.compute.amazonaws.com'
node {
    stage('Code Pickup') {
    echo "Source Code Repository Type : ${CodeType}"
    echo "Source Code Repository Path : ${CodeLoc}"
    
    if("${CodeType}".toUpperCase()=='SVN'){
        sh "svn co --username ${scmUsername} --password ${scmPassword} ${CodeLoc} ."
        
    } else if("${CodeType}".toUpperCase()=='GIT'){
        CodeLoc = CodeLoc.substring(0, CodeLoc.indexOf("//")+2) + scmUsername + ":" + scmPassword + "@" +CodeLoc.substring(CodeLoc.indexOf("//")+2, CodeLoc.length());
        try {
            sh 'ls -a | xargs rm -fr' 
        } catch (error) {
        }
        sh "git clone ${CodeLoc} ."        
    } else {
        error 'Unknown Source code repository. Only GIT and SVN are supported'
    }
  } 
//END OF CODE PICKUP STAGE
//_______________________________________________________________________________________________________________________________________________________________________  
//BUILD & PACKAGE
def appModuleSeperated = fileExists 'app'
def testModuleSeperated = fileExists 'test'
def appPath = ''
def testPath = ''  
if (appModuleSeperated) {
    echo 'App Module is found , assumed that application is present in /app directory'
    appPath='app/'
} else {
    echo 'There is no defined Application path , hence it is assumed that application is in current directory'
    appPath = ''
}

if (testModuleSeperated) {
    echo 'Test Module is found , assumed that Test Cases are Present for the concerned Modules and has to be performed'
    testPath = 'test/'
} else {
    echo 'No Test Modules found , hence it is assumed that no test environment and / or test cases to be performed'
    testPath = ''
}
def isBuildAndPackageRequired = true
def buildDockerFile = appPath + 'Dockerfile.build'
def distDockerFile = appPath + 'Dockerfile.dist'
if (fileExists(buildDockerFile) && fileExists(distDockerFile)) {
    echo 'Looks like this application contains seperate Build and Distribution Docker Files , its assumed to be developed in compiler dependant programming language.'
    isBuildAndPackageRequired = true;    
} else if (appPath + fileExists('Dockerfile')) {
    echo 'This application contains a single docker file , it is assumed to be developed in compiler in-dependant / interpreter based programming language.'
    isBuildAndPackageRequired = false;
    distDockerFile = appPath + 'Dockerfile'
} else {
    echo 'Dockerfile not found under ' + appPath
  }
// COPYING APP Directory to Current Working Directory
  def appWorkingDir = (appPath=='') ? '.' : appPath.substring(0, appPath.length()-1)  
// NEXUS file for Time Stamp comparison. This file is used for comparing time stamps and differentiating input files from generated output files.        
    sh 'echo Nexus>Nexus.txt'
//END OF INITIALIZING.
//_______________________________________________________________________________________________________________________________________________________________________  
//BUILD & PACKING
if(isBuildAndPackageRequired){
    echo 'Compile and Packaging runs as separate entities , it is inferred that the App package is compiler dependant.'
    stage('Compile, Unit Test & Package') {
      echo 'Working Directory for Docker Build file: ' + appWorkingDir
      echo "Build Tag Name: ${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}"
      echo "Build params: --file ${buildDockerFile} ${appWorkingDir}"
      
      appCompileAndPackageImg = docker.build("${dockerRepo}/${dockerImageName}-build:${env.BUILD_NUMBER}", "--file ${buildDockerFile} ${appWorkingDir}")      
      
def dockerCMD = readFile buildDockerFile
echo dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())      
appCompileAndPackageImg.inside('--net=host') {        
sh dockerCMD.substring(dockerCMD.indexOf('CMD')+3, dockerCMD.length())
}
}
} else {
echo 'Compile and Package are not seperate steps , it is inferred that the package is not compiler dependant'
}
  //---------------------------------------
  
  if("${stage}".toUpperCase() == 'BUILD') {
    echo 'It is inferred that the package is a Build only application , hence it is moved to a temporary repository'
    docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Stage') {
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push();
      }
    }   
  } else if ("${stage}".toUpperCase() == 'DEPLOY') {
    echo 'It is inferred that the package is a deploy only application , hence it has to be moved to a permanent repository'
    docker.withRegistry("https://${permanentDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Dockerization & Publish') {
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push('latest');
      }
    }    
  } else if ("${stage}".toUpperCase() == 'CERTIFY'){
    echo 'It is inferred that the package is a certify only application , hence it has to be moved to a provisioned with a runtime sandbox environment and push it to temporary repository'
    docker.withRegistry("http://${temporaryDockerRegistry}/", 'docker-registry-login') {
      def pcImg
      stage('Certify') {
        pcImg = docker.build("${dockerRepo}/${dockerImageName}:${env.BUILD_NUMBER}", "--file ${distDockerFile} ${appWorkingDir}")
        pcImg.push('SNAPSHOT');
      }
    }   
  }
//END OF IMAGE PUSHING INTO REPOSITORY
// NEXUS UPDATE
  stage('Publish Jenkins Output to Nexus'){
        echo 'Publishing the artifacts...';
        sh 'rm Nexus.txt'
//NEXUS FLOW ENDS HERE
  }
}
