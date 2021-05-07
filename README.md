# Jenkins_CI_scripted_pipiline
Jenkins_CI_scripted_pipiline
############################################
node{
     stage("SCM"){
        git branch: 'main', url: 'https://github.com/cloudtechmasters/springboot-docker-assignment-without-database.git'
     }
     //Here if we want we can use parameterized repo url and branch name as well
     //stage("SCM"){
     //   git branch: "${params.Branch_Name}", url: "${params.GIT_URL}"
     //}
     stage("Build Artifact") {
         sh "mvn clean package"
     }
	 stage("Sonar Analysis") {
	    
	    def HOST= "http://34.204.204.27:9000"
	    def Project= "test"
	    sonar_token = "422207d3c02f5754b6210a8aa1b2d573673aa7a3"
		
		    sh "mvn test"     
            sh "mvn sonar:sonar -Dsonar.login=$sonar_token -Dsonar.host.url=$HOST -Dsonar.projectKey=$Project"
            sh "sleep 5"
            sh "curl $HOST/api/qualitygates/project_status?projectKey=$Project >result.json"
            sh "cat result.json"
            sh 'if [ $(jq -r \'.projectStatus.status\' result.json) = OK ] ; then $CODEBUILD_BUILD_SUCCEEDING -eq 0 ;fi'
	 }
	 stage("Publish to Nexus Repository Manager") {
		def VERSION = sh(script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout', returnStdout: true)
        def GROUPID = sh(script: 'mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout', returnStdout: true)
        def ARTIFACTID = sh(script: 'mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout', returnStdout: true)
        def PACKAGING = sh(script: 'mvn help:evaluate -Dexpression=project.packaging -q -DforceStdout', returnStdout: true)
		def HOST= "http://34.204.204.27:8081"
		
	    if (VERSION.contains("SNAPSHOT") ) {
    	    sh(script: 'mvn deploy:deploy-file -DgroupId=$GROUPID \
            -DartifactId=$ARTIFACTID \
            -Dversion=$VERSION \
            -Dpackaging=$PACKAGING \
            -Dfile=target/$ARTIFACTID-$VERSION.$PACKAGING \
            -DrepositoryId=nexus-snapshots \
            -Durl=$HOST/repository/maven-snapshots/', returnStdout: true)
	    } else {
	        sh(script: 'mvn deploy:deploy-file -DgroupId=$GROUPID \
            -DartifactId=$ARTIFACTID \
            -Dversion=$VERSION \
            -Dpackaging=$PACKAGING \
            -Dfile=target/$ARTIFACTID-$VERSION.$PACKAGING \
            -DrepositoryId=nexus-releases \
            -Durl=$HOST/repository/maven-releases/', returnStdout: true)
	    }
	}
}
