pipeline {
    agent any
	
	parameters {
            
               choice(choices: 'build\ndeploy-to-dev\ndeploy-proxy\ndeploy-to-uat', description: 'Which Env', name: 'ENV_DEPLOY')
               
               string(name: 'ARTIFACT_VERSION', defaultValue: '', description: 'Enter Artifact version from Artifactory.')
               
           }
	
    stages {
		stage('SampleJob - Checkout') {
			when { expression { params.ENV_DEPLOY == 'all' } }
            steps {
                echo 'Fetching App from Git repo'
					checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git-repo', url: '${gitCommon}/'+params.GIT_REPO_NAME+'.git']]])
            }
        }
        stage ('SampleJob - Build') {
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
		//when { expression { params.ENV_PROMOTE == 'all' } }
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

					  //buildInfo.retention maxBuilds: 10, maxDays: 7, deleteBuildArtifacts: true
					  // Publish build info.
					  artifactory_server.publishBuildInfo buildInfo
					
				}
			
			}
		}
		stage ('Uploading to PCF') {
            steps {
                echo 'Uploading app...'
				script {
					pushToCloudFoundry cloudSpace: 'bcbsma', credentialsId: 'pcf-cre', organization: 'Northeast / Canada', target: 'https://api.run.pivotal.io'
				}
            }
        }
	}
}