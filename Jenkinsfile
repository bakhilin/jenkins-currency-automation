@Library('jenkins-shared-library') _
import org.currency.DockerUtils

node('built-in'){
    try {   
        if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('n.bakhilin')) {
            def dockerUsername = 'bakhilin'
            def dockerImageTag = '0.1.1'
            def dockerImageName = "${dockerUsername}/currency-rest-api"
            def buildSuccess = true
            def dockerUtils = new DockerUtils(this)
        
            stage('CLONE repo') {
                buildSuccess = pipeline_utils.errorHandler(STAGE_NAME){
                    checkout scm
                }
            }

            stage('LINT dockerfile') {
                try {
                    sh 'hadolint Dockerfile'
                    echo "Dockerfile is valid"
                } catch(err) {
                    echo "Linting errors: ${err.getMessage()}"
                    currentBuild.result = 'UNSTABLE'
                }
            }

            stage('BUILD Docker Image') {
                if (buildSuccess) {
                    buildSuccess = pipeline_utils.errorHandler(STAGE_NAME){
                        dockerUtils.buildImage(dockerImageName, dockerImageTag)
                    }
                }
            }

            stage('SCAN and LOGIN') {
                parallel(
                    'SCAN image by Trivy': {
                        dockerUtils.scanImageTrivy(dockerImageName, dockerImageTag)
                    }, 
                    'Login to Docker Hub': {
                        if (buildSuccess) {
                            buildSuccess = pipeline_utils.errorHandler(STAGE_NAME) {
                                dockerUtils.loginToDockerHub('docker-hub-credentials')
                            }
                        }
                    }
                )
            }

            stage('DEPLOY and DELETE') {
                parallel(
                    'DEPLOY Docker Image': {
                        if (buildSuccess || currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                            buildSuccess = pipeline_utils.errorHandler(STAGE_NAME) {
                                dockerUtils.pushImageDockerHub(dockerImageName, dockerImageTag)
                            }   
                        }                        
                    },
                    'DELETE bad images': {
                        pipeline_utils.errorHandler(STAGE_NAME){
                            dockerUtils.deleteImages()
                        }
                    }
                )
            }
        } else {
            echo "Branch name not started with n.bakhilin or master, your: ${env.BRANCH_NAME}"
        }
    } catch(err) {
        echo "Caught: ${err}"
    } finally {
        pipeline_utils.logBuildInfo()
    }
}

