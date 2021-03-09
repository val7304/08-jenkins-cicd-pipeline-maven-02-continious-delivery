pipeline{
    agent    any    
    tools  {
        maven "my-maven-3"
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "host.docker.internal:8081"
        NEXUS_REPOSITORY_SNAPSHOT  = "mylocalrepo-snapshots"
        NEXUS_REPOSITORY_RELEASE = "mylocalRepo-release"
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"    
    }
    
    stages{
        // Clone from Git
        stage("Clone App from Git"){
            steps{
                echo "====++++  Clone App from Git ++++===="
                git branch:"master", url: "https://github.com/val7304/08-jenkins-cicd-pipeline-maven-02-continious-delivery.git"
            }          
        }
        // Build and Unit Test (Maven/JUnit)
        stage("Build and Unit Test (Maven/JUnit)"){
            steps{
                echo "====++++  Build and Unit Test (Maven/JUnit) ++++===="
                sh "mvn clean package"
                junit '**/target/surefire-reports/TEST-*.xml' // Archive the test reports
            }           
        }
        // Static Code Analysis (SonarQube)        
        stage("Static Code Analysis (SonarQube)"){
            steps{
                echo "====++++  Static Code Analysis (SonarQube) ++++===="
                withSonarQubeEnv('my_sonarqube_in_docker') {  
																																																																																																																																			
                   sh "mvn clean package sonar:sonar -Dsurefire.skip=true  -Dsonar.host.url=http://host.docker.internal:9000   -Dsonar.projectName=08-jenkins-cicd-pipeline-maven-01-continious-delivery -Dsonar.projectKey=08-jenkins-cicd-pipeline-maven-01-continious-delivery -Dsonar.projectVersion=$BUILD_NUMBER";
                }  
            }           
        }
        stage("Checking the Quality Gate") {
            steps {
                echo "====++++  Checking the returned SonarQube Quality Gate ++++===="
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }        
        
        // Integration Test (Maven/JUnit)
        stage("Integration Test (Maven/JUnit)"){
            steps{
                echo "====++++  Integration Test (Maven/JUnit) ++++===="
             //   sh "mvn clean verify -Dsurefire.skip=true"    // Do not repeat the unit tests, they have been already done
              sh "mvn clean package -Dsurefire.skip=true failsafe:integration-test"
              junit "**/target/failsafe-reports/*.xml" // Archive the test reports
            }         
        }
       
	          
        // Publication de l'artifact (Snaphot) sur Nexus3
        stage("Publish to Nexus Repository Manager") {
            steps {
                echo "====++++  Publish to Nexus Repository Manager ++++===="
			
				script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
																																														  
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY_SNAPSHOT,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        
        // Start the Staging Server
        stage ('Start the Staging Server ') {
	        steps {
               echo "====++++  Start the Staging Server ++++===="
	           sh "docker run --name staging_server -p 9090:8080  -d tomcat:9.0"
	        }
	    }
        
        //Deploy the app on the Staging Server
        stage ('Deploy the app on the staging ') {
	        steps {
               // TODO  - GET THE war artifact from Nexus
               echo "====++++  Deploy the war on the Staging Server ++++===="
	           sh "docker cp $WORKSPACE/target/greetings-0.1-SNAPSHOT.war staging_server:/usr/local/tomcat/webapps/hello.war"
            }
	    }
        // Do performance tests on the app
	    stage('Do the Performance Testing'){	   	   
	   	    steps {  
               echo "====++++ Do the performance tests Against the Staging Server ++++===="        
               sh "mvn clean verify"   // This triggers JMeter Performance Analysis and generates results in target/jmeter/results          
               sh "ls ${WORKSPACE}/target/jmeter/results"
         	}
	    }
        // Publish the test reports
	    stage ('Publish the performance reports') {
	        steps {
               echo "====++++ Publish the performance Reports and Check the Threasholds ++++===="   
	           perfReport sourceDataFiles: "$WORKSPACE/target/jmeter/results/*.csv"	     
	        }
	    }	    
     
		
		// Promote the app On NEXUS from the SNAPSHOT -> RELEASE
		stage("Promote the app in  Nexus Repository Manager") {
			steps {
				echo "====++++  Promote the app in  Nexus Repository Manager ++++===="
				sh "mv target/greetings-0.1-SNAPSHOT.war target/greetings-0.1-RELEASE.war"
            script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;                                                  
                    artifactVersion ="0.1-RELEASE-${BUILD_NUMBER}";  //Append the build number for traceability
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: artifactVersion,
                            repository: NEXUS_REPOSITORY_RELEASE,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }  

	   

		stage ('Start the Deployment  Server ') {
	        steps {
               echo "====++++  Start the Staging Server ++++===="
	           sh "docker run --name deployment_server -p 9080:8080  -d tomcat:9.0"
	        }
	    }
        
        //Deploy the app on the Staging Server
        stage ('Deploy the app on the Deployment Server ') {
                
            steps {
               // TODO  - GET THE war artifact from Nexus
               echo "====++++  Deploy the war on the Deployment Server ++++===="

	           sh "docker cp $WORKSPACE/target/greetings-0.1-RELEASE.war deployment_server:/usr/local/tomcat/webapps/hello.war"
            }
	    }     
        
    }   
}
