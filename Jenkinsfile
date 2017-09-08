#!groovy
@Library('savvi-jenkins-pipeline-libraries@test_branch') _

pipeline {
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'develop', description: 'What branch to build?')
    }

    agent {
        label 'Savvi-Build-Slave'
    }

    triggers {
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
    }

    post {
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'
        }
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        always {
            sendSavviNotifications currentBuild.result
        }
    }

    stages {
        stage('Initialise') {
            steps {
                deleteDir()
                updateGitlabCommitStatus name: 'build', state: 'pending'

                // Code that doesn't have a step plugin equivalent
                script {
                    GIT_BRANCH = params.GIT_BRANCH
                    // GitLab plugin sets env.gitLabSourceBranch so use this if its a request from GitLab
                    if (env.gitLabSourceBranch) GIT_BRANCH = env.gitLabSourceBranch

                    // Populate the final choice to environment so our scripts can use it
                    env.GIT_BRANCH = GIT_BRANCH

                    artifactoryServer = Artifactory.server('apro.nbnco.net.au')
                }

                sh 'env'
            }
        }

        stage('Preparation') {
            steps {
                dir("${JOB_BASE_NAME}") {
                    git(
                        branch: GIT_BRANCH,
                        credentialsId: 'snpm_git_user',
                        url: "https://git.nbnco.net.au/snpm/${JOB_BASE_NAME}.git"
                    )

                    script {
                        env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                        env.ROOT_BUILD_CAUSE = java.net.URLEncoder.encode(currentBuild.rawBuild.getCauses()[0].getShortDescription(), "UTF-8")
                    }

                    // Make sure everythign env-wise is set before this if its used in notifications.
                    sendSavviNotifications 'STARTED'
                }
            }
        }

        stage('Dependencies') {
            steps {
                script {
                    def downloadSpec = """{
                        "files": [
                            {
                                "pattern": "savvi-packages/savvi/6/release/x86_64/savvi-node6-6.10.1-0.*",
                                "target": "dependencies/",
                                "flat": "true"
                            },
                            {
                                "pattern": "savvi-packages/savvi/6/develop/x86_64/savvi-nodejs-utils-1.0.0~2-*",
                                "target": "dependencies/",
                                "flat": "true"
                            }
                        ]
                    }"""
                    artifactoryServer.download(spec: downloadSpec)
                }

                // God I hate yum for returning exit code 1 when something is already installed (from file)
                sh 'sudo rpm -qa | grep savvi-node6 || sudo yum install -y dependencies/savvi-node6-*'
                sh 'sudo rpm -qa | grep savvi-nodejs-utils || sudo yum install -y dependencies/savvi-nodejs-utils-*'
            }
        }

        stage('Tests') {
            steps {
                script {
                    env.PATH="/opt/savvi-deps/node6/bin:${env.PATH}"
                }
                dir("$WORKSPACE/${JOB_BASE_NAME}") {
                    sh 'which node'
                    sh './tests/unit-tests/unit-tests.sh'
                }
            }
            post {
                success {
                    junit '**/*.xml'
                }
            }
        }

        stage('Build') {
            steps {
                dir("${JOB_BASE_NAME}") {
                    sh './packaging/rpm/build.bash'
                }
            }
        }

        stage('Package') {
            steps {
                dir('savvi-build-tools-nbn') {
                    git(
                        branch: 'develop',
                        credentialsId: 'snpm_git_user',
                        url: 'https://git.nbnco.net.au/snpm/savvi-build-tools-nbn.git'
                    )
                }

                sh './savvi-build-tools-nbn/scripts/rpmbuilder.sh'
            }
        }

        stage('Publish') {
            steps {
                script {
                    def uploadSpec = readFile 'uploadfile.spec'
                    buildInfo = artifactoryServer.upload(spec: uploadSpec)
                    artifactoryServer.publishBuildInfo(buildInfo)

                    // Could be useful...
                    buildInfoJson = getArtifactoryBuildInfo(buildInfo)

                    // This artifact had unit tests during build, so mark the build artifacts as passed
                    def artifacts = getArtifactoryBuildArtifacts(buildInfo)
                    artifacts.results.each {
                        updateArtifactoryProperty {
                            url = it.apiStorageUri
                            credentialsId = 'apro_savvi_user'
                            key = 'unit'
                            value = 'true'
                        }
                    }

                    // Put a string list of artifacts produced into an env variable so our scripts can see
                    env.artifacts = buildInfo.deployedArtifacts.collect{ it.getName() }.join(' ')
                }
            }
        }

        stage('Deploy') {
            steps {
                build(
                    wait: false,
                    job: 'savvi-fake-product-deploy',
                    parameters: [
                        [$class: 'StringParameterValue', name: 'UT_JOB_NAME', value: JOB_NAME],
                        [$class: 'StringParameterValue', name: 'UT_BUILD_NUMBER', value: BUILD_NUMBER],
                        [$class: 'StringParameterValue', name: 'DEPLOY_VERSION', value: 'jenkins-latest']
                    ]
                )
            }
        }
    }
}