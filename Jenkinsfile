pipeline {
    agent any
	
	parameters{
		choice(choices:'build\ndeploy-to-dev\ndeploy-proxy\ndeploy-to-uat',description:'Which Env',name:'ENV_DEPLOY')
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
        when { expression { params.ENV_DEPLOY == 'build' } }
            steps {
                echo 'Building app...'
				script {
					withMaven(maven: 'maven') { 
						if(isUnix()) {
							sh "mvn clean install -DskipTests " 
						} else { 
							bat "mvn clean install -DskipTests " 
						} 
					}
				}
            }
        }
        stage ('SampleJob - Unit Tests') {
        agent any
        when { expression { params.ENV_DEPLOY == 'build' } }
            steps {
                echo 'Building app...'
				script {
					withMaven(maven: 'maven') { 
						if(isUnix()) {
							sh "mvn clean install -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false " 
						} else { 
							bat "mvn clean install -DskipTests -Djacoco.skip=false -Djacoco.skip.report=false "  
						} 
					}
				}
            }
        }
		stage('Upload to Artifactory') {
		agent any
		when { expression { params.ENV_DEPLOY == 'build' } }
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

					  rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo

					  
					  artifactory_server.publishBuildInfo buildInfo
					
				}
			
			}
		}
		stage ('Uploading to PCF') {
		agent any
		when { expression { params.ENV_DEPLOY == 'build' } }
            steps {
                echo 'Uploading app...'
				script {
					pushToCloudFoundry cloudSpace: 'bcbsma', credentialsId: 'pcf-cre', organization: 'Northeast / Canada', target: 'https://api.run.pivotal.io'
				}
            }
        }
        stage ('Deploy Proxy') {
        agent any
	        when { expression { params.ENV_DEPLOY == 'deploy-proxy' } }
	            steps {
	                echo 'Deploying proxy...'
					script {
						withCredentials([usernamePassword(credentialsId: 'apigee-cred', passwordVariable: 'SECREAT_APIGEE_PASSWORD', usernameVariable: 'SECREAT_APIGEE_USER')]) {
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
            steps {
                echo 'Updating KVM...'
				//script {
					//pushToCloudFoundry cloudSpace: 'bcbsma', credentialsId: 'pcf-cre', organization: 'Northeast / Canada', target: 'https://api.run.pivotal.io'
				//}
            }
        }
	}
}