#!groovy
import groovy.json.JsonSlurperClassic
node {
    def branchName
    def lastID
    def latestID
    def deploymentHistoryBranchName = 'deployment'
    def TEST_LEVEL = 'RunLocalTests'    
    def commitFileName = "${env.JOB_NAME}.txt"
    def commitFilePath = commitFileName.replaceAll('/','\\\\')
    def BUILD_NUMBER=env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
    
    
    def SF_CONSUMER_KEY=env.SF_CONSUMER_KEY
    def SF_USERNAME=env.SF_USERNAME
    def SERVER_KEY_CREDENTIALS_ID=env.SERVER_KEY_CREDENTIALS_ID
    def DEPLOYDIR='src'
    def TEST_LEVEL='RunLocalTests'
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://test.salesforce.com"


    def toolbelt = tool 'toolbelt'
   
    println commitFilePath
    println consumer_key
    println server_key_id
    println user_name
    println instance_url
    println commitFileName
    //
    // -------------------------------------------------------------------------
    // Check out code from source control GIT
    // -------------------------------------------------------------------------

    stage('Clean Workspace') {
        try {
            deleteDir()
        }
        catch (Exception e) {
            println('Unable to Clean WorkSpace.')
        }
    }

    stage('Checkout Git Source') {
        checkout scm
        
        def branchOriginName = bat (label: 'Branch name', script: '@git name-rev --name-only HEAD', returnStdout: true).trim() as String   
        branchName = branchOriginName.replaceAll('remotes/origin/','').split('~')[0]
        println branchName
        
    }

    // -------------------------------------------------------------------------
    // Run all the enclosed stages with access to the Salesforce
    // JWT key credentials.
    // -------------------------------------------------------------------------
    withEnv(["HOME=${env.WORKSPACE}"]) {	
	
	    withCredentials([file(credentialsId: SERVER_KEY_CREDENTIALS_ID, variable: 'server_key_file')]) {
		// -------------------------------------------------------------------------
		// Authenticate to Salesforce using the server key.
		// -------------------------------------------------------------------------

		stage('Authorize to Salesforce') {
			rc = command "${toolbelt}/sfdx auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --jwtkeyfile ${server_key_file} --username ${SF_USERNAME} --setalias UATSFDX"
		    if (rc != 0) {
			error 'Salesforce org authorization failed.'
		    }
		}

        stage('Get Last Deployment'){
            script{
                println branchName
                bat 'git checkout ' + deploymentHistoryBranchName + ''
                
                lastID = bat(label: 'Get last commit id', returnStdout: true, script:'@powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"').trim() as String
                //def lastID = bat 'powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"'
                println lastID

                bat 'git checkout ' + branchName + ''
            }            
        }

        //Run Powershell Sript - CHANGE HERE
        stage('Delta Deployment') {
            script {
                
                // def branchOriginName = bat (label: 'Branch name', script: '@git name-rev --name-only HEAD', returnStdout: true).trim() as String   
                // def branchName = branchOriginName.replaceAll('remotes/origin/','').split('~')[0]
                // println branchName
                
                //def lastID = bat(label: 'Get last commit id', returnStdout: true, script:'@powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"').trim() as String
                //def lastID = bat 'powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"'
                println lastID
                bat 'powershell -command "git diff-tree --no-commit-id --name-only -r ' + lastID + ' head > list.txt"'
                bat 'python copyDeltaFiles.py'

                try {
                    bat 'groovy PackageXMLGenerator.groovy delta/force-app/main/default delta/force-app/main/default/package.xml'
                    bat 'sfdx force:mdapi:deploy -d delta/force-app/main/default -w 30 --targetusername ' + SF_USERNAME + ''
                    // latestID = bat(label: 'Get latest commit id', returnStdout: true, script:'@git rev-parse HEAD').trim() as String
                    // println latestID
                    //bat 'git rev-parse HEAD > DevOps\\LastDeploymentState\\' + commitFilePath + ' '
                    // // Git commit latest deployed commitid
                    // //bat 'git commit ' + commitFileName + ' -m "Delta Deployment succeeded" '
                    // def latestID = bat(label: 'Get latest commit id', returnStdout: true, script:'@powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"').trim() as String
                    // println latestID
                    // //bat 'git config core.autocrlf true'
                    // bat 'git commit -am "Delta Deployment succeeded" '
                    // //bat 'git remote show origin'
                    // bat 'git push ' + git_repository_url + ' HEAD:' + branchName + ' --force'

                } catch (Exception e) {
                    throw e
                //println("No files for Delta deployment")
                }
                
                try {
                    bat 'groovy DestructiveXMLGenerator.groovy deltaDestruction/force-app/main/default deltaDestruction/force-app/main/default/manifest/destructiveChanges.xml'
                    bat 'sfdx force:mdapi:deploy -d deltaDestruction/force-app/main/default/manifest -w 30 --targetusername ' + SF_USERNAME + ''
                    //bat 'git rev-parse HEAD > DevOps\\LastDeploymentState\\' + commitFilePath + ' '                    
                    
                } catch (Exception e) {
                    println("No files for Delta destruction deployment")
                } 

                latestID = bat(label: 'Get latest commit id', returnStdout: true, script:'@git rev-parse HEAD').trim() as String
                println latestID
            }
        }

        
        stage('Set Latest Deployment'){
            script{
                println branchName
                println latestID
                bat 'git checkout ' + deploymentHistoryBranchName + ''

                //bat '' + latestID + ' > DevOps\\LastDeploymentState\\' + commitFilePath + ' '
                bat(label: 'Set file content', returnStdout: true, script:'@powershell -command "Set-Content -Path DevOps\\LastDeploymentState\\' + commitFilePath + ' -Value ' + latestID + ' -Force "')
                // Git commit latest deployed commitid
                //bat 'git commit ' + commitFileName + ' -m "Delta Deployment succeeded" '
                def latest = bat(label: 'Get latest commit id', returnStdout: true, script:'@powershell -command "cat DevOps\\LastDeploymentState\\' + commitFilePath + '"').trim() as String
                println latest
                //bat 'git config core.autocrlf true'
                bat 'git commit -am "Delta Deployment succeeded" '
                //bat 'git remote show origin'
                bat 'git push ' + git_repository_url + ' HEAD:' + deploymentHistoryBranchName + ' --force'
            }            
        }

/* 
        // -------------------------------------------------------------------------
        // Example shows how to run a check-only deploy.
        // -------------------------------------------------------------------------
        stage('Check Only Deploy') {
            script {
                try {
                    rc = command "${toolbelt}/sfdx force:mdapi:deploy -d delta/force-app/main/default -w 30 --targetusername ${user_name}"
                    if (rc != 0) {
                        error 'Salesforce deploy failed.'
                    }
                    else{
                        // Git commit latest deployed commitid
                        bat 'git commit ${commitFileName} -m "Delta Deployment succeeded" '
                        bat 'git push '
                    }
                } catch (Exception e) {
                    println("No Delta deployment")
                }

            }
        }
            */
   	 }
    }
}

def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}
