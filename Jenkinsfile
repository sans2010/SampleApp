#!/usr/bin/env groovy

import groovy.json.JsonSlurper

pipeline {
    agent any
	
	parameters{
		choice(choices:'deploy-to-dev\ndeploy-proxy-dev\ndeploy-kvm-dev',description:'Which Env',name:'ENV_DEPLOY')
		string(name:'ARTIFACT_VERSION',defaultValue:'',description:'Enter Artifact version from Artifactory.')
	}
	
    stages {
		//stage('SampleJob - Checkout') {
			//when { expression { params.ENV_DEPLOY == 'all' } }
            //steps {
                //echo 'Fetching App from Git repo'
					//checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git-repo', url: '${gitCommon}/'+params.GIT_REPO_NAME+'.git']]])
            //}
       // }
        stage ('SampleJob - Build') {
        agent any
        when { expression { params.ENV_DEPLOY == 'deploy-to-dev' } }
            steps {
                echo 'Building app...'
				script {
					mvnHome = tool 'maven'
					//withMaven(maven: 'maven') { 
						if(isUnix()) {
							sh "mvn clean install -DskipTests " 
						} else { 
							bat "${mvnHome}/bin/mvn clean install -DskipTests " 
						} 
					//}
				}
            }
        }
        stage ('SampleJob - Unit Tests') {
        agent any
        when { expression { params.ENV_DEPLOY == 'deploy-to-dev' } }
            steps {
                echo 'Building app...'
				script {
					mvnHome = tool 'maven'
					//withMaven(maven: 'maven') { 
						if(isUnix()) {
							//sh "mvn clean generate-resources -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false "
							sh "mvn clean test -Djacoco.skip=false -Djacoco.skip.report=false " 
						} else { 
							//bat "mvn clean generate-resources -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false "
							bat "${mvnHome}/bin/mvn clean test -Djacoco.skip=false -Djacoco.skip.report=false "  
							jacoco()
						}
						println "WORKSPACE = " + WORKSPACE
						//junit '$WORKSPACE/target/surefire-reports/*.xml' 
					//}
					println "WORKSPACE = " + WORKSPACE
					//junit '$WORKSPACE/target/surefire-reports/*.xml'
				}
            }
        }
		stage('Upload to Artifactory') {
		agent any
		when { expression { params.ENV_DEPLOY == 'deploy-to-dev' } }
			steps {
				echo 'Building app...'
				script{
					def artifactory_server = Artifactory.server "Artifactory"
					def buildInfo = Artifactory.newBuildInfo()
					buildInfo.env.capture = true
					def rtMaven = Artifactory.newMavenBuild();
					rtMaven.tool = "maven"
					
					  rtMaven.deployer releaseRepo:'libs-release-local', snapshotRepo:'libs-snapshot-local', server: artifactory_server
					  rtMaven.resolver releaseRepo:'libs-release', snapshotRepo:'libs-snapshot', server: artifactory_server

					  rtMaven.run pom: 'pom.xml', goals: 'clean install -DskipTests', buildInfo: buildInfo

					  
					  artifactory_server.publishBuildInfo buildInfo
					//junit '/target/surefire-reports/*.xml'
				}
			
			}
		}
		stage ('Uploading to PCF') {
		agent any
		when { expression { params.ENV_DEPLOY == 'deploy-to-dev' } }
            steps {
                echo 'Uploading app...'
				script {
					pushToCloudFoundry cloudSpace: 'bcbsma', credentialsId: 'pcf-cre', organization: 'Northeast / Canada', target: 'https://api.run.pivotal.io'
				}
            }
        }
        stage ('Deploy Proxy') {
        agent any
	        when { expression { params.ENV_DEPLOY == 'deploy-proxy-dev' } }
	            steps {
	                echo 'Deploying proxy...'
					script {
						withCredentials([usernamePassword(credentialsId: 'apigee-sandesh', passwordVariable: 'SECREAT_APIGEE_PASSWORD', usernameVariable: 'SECREAT_APIGEE_USER')]) {
							withMaven(maven: 'maven') { 
								if(isUnix()) {
									sh "mvn clean install -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false " 
								} else { 
									bat "mvn install -f apigee/apigee-pom.xml -Pdev -Dusername=$SECREAT_APIGEE_USER -Dpassword=$SECREAT_APIGEE_PASSWORD -Dorg=bcbsma -Doptions=validate " 
								} 
						    
						    
						}
						/* withMaven(maven: 'maven') { 
							if(isUnix()) {
								sh "mvn clean install -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false " 
							} else { 
								bat "mvn install -P $ENVIRONMENT -Dusername=$APIGEE_USERNAME -Dpassword=$APIGEE_PASSWORD -Dorg=bcbsma -Doptions=validate "  
							} 
						} */
					}
	            }
	        }
        }
        stage ('KVM Updated') {
        agent any
	        when { expression { params.ENV_DEPLOY == 'deploy-kvm-dev' } }
            steps {
                echo 'Updating KVM...'
				script {
					def secretText="U2FuZGVzaC5HYXdhbGlAcGVyZmljaWVudC5jb206QXBpZ2VlQDIwMTk="
					def filename="microgateway-router"
					def jsonSlurper = new JsonSlurper()
					def reader = new BufferedReader(new InputStreamReader(new FileInputStream("$WORKSPACE/apigee/microgateway-router.json"),"UTF-8"))
    				def data = jsonSlurper.parse(reader)
    				def entryName = data.name
    				jsonSlurper = null
    				data = null
    				reader = null
					//def URL="https://api.enterprise.apigee.com/v1/organizations/bcbsma/environments/$ENVIRONMENT/keyvaluemaps/microgateway-router/entries/$entryName"
					def URL="https://api.enterprise.apigee.com/v1/organizations/bcbsma/environments/dev/keyvaluemaps/microgateway-router/entries/${entryName}"
					def URL1 = "https://api.enterprise.apigee.com/v1/organizations/bcbsma/environments/dev/keyvaluemaps/microgateway-router/entries"
				    					
					bat 'curl -S -k --silent -X GET --header "Authorization: Basic '+secretText+'" '+URL+' --write-out "HTTPSTATUS=%%{http_code}" > result.txt'
					applyKvm(secretText, filename, URL, URL1)
				}
            }
        }
	}
}

def applyKvm(String secretText, String filename, String URL, String URL1) {
	def HTTP_RESPONSE = readFile 'result.txt'
	//def HTTP_BODY = (HTTP_RESPONSE =~ 'HTTPSTATUS=[0-9]{3}$')
	//println "HTTP_BODY = " + HTTP_BODY[0]
	//def HTTP_BODY = ~'s/HTTPSTATUS\:[0-9]{3}$//'
	//println HTTP_BODY.size()
	//def HTTP_RESPONSE_BODY = (HTTP_RESPONSE - HTTP_BODY[0])
	def HTTP_CODE = HTTP_RESPONSE.substring(HTTP_RESPONSE.lastIndexOf("=")+1)
	println "HTTP_CODE = " + HTTP_CODE
	println "filename = " + filename
	String fileContents = new File(''+WORKSPACE+'/apigee/microgateway-router.json').text
	//def routerContent = readFile 'apigee/microgateway-router.json'
	fileContents = fileContents.replaceAll("\\r\\n|\\r|\\n", " ");
	fileContents = fileContents.replace("\n", "").replace('\"', '\\"');
	//fileContents = '{"name": "edgemicro_sample_app_1","value": "https://mocktarget.apigee.net"}'
	println "fileContents = " + fileContents
	if (HTTP_CODE == "200") {
		//update an entry that already exist
		bat 'curl -S -k --silent -X POST --header "Content-Type: application/json" --header "Authorization: Basic '+secretText+'" --data "'+fileContents+'" '+URL+' --write-out "HTTPSTATUS1=%%{http_code}" > result1.txt'
		String UPDATE_RESPONSE = new File('result1.txt').text
		//def UPDATE_RESPONSE = readFile 'result.txt'
		println "UPDATE_RESPONSE = " + UPDATE_RESPONSE
	} else {
		//entry doesn’t exist create new entry
		println "1151"
		bat 'curl -S -k --silent -X POST --header "Content-Type: application/json" --header "Authorization: Basic '+secretText+'" --data "'+fileContents+'" '+URL1+' --write-out "HTTPSTATUS2=%%{http_code}" > result2.txt'
		String CREATE_RESPONSE = new File('result2.txt').text
		println "CREATE_RESPONSE = " + CREATE_RESPONSE
	}
}