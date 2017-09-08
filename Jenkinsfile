#!groovy

pipeline {
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'What branch to build?')
    }


    agent {
        label 'master'
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
    }

    post {
        always {
            echo 'This will always run'
        }
        success {
            echo 'This will run only if successful'
        }
        failure {
            echo 'This will run only if failed'
        }
        unstable {
            echo 'This will run only if the run was marked as unstable'
        }
        changed {
            echo 'This will run only if the state of the Pipeline has changed'
            echo 'For example, if the Pipeline was previously failing but is now successful'
        }
    }

    stages {
        stage('Initialise') {
            steps {
                deleteDir()
                //updateGitlabCommitStatus name: 'build', state: 'pending'

                // Code that doesn't have a step plugin equivalent
                script {
                    GIT_BRANCH = params.GIT_BRANCH
                    // GitLab plugin sets env.gitLabSourceBranch so use this if its a request from GitLab
                    if (env.gitLabSourceBranch) GIT_BRANCH = env.gitLabSourceBranch

                    // Populate the final choice to environment so our scripts can use it
                    env.GIT_BRANCH = GIT_BRANCH

                    //artifactoryServer = Artifactory.server('apro.nbnco.net.au')
                }

                sh 'env'
            }
        }

        stage('Preparation') {
            steps {
                dir("${JOB_BASE_NAME}") {
                    git(
                        branch: GIT_BRANCH,
                        credentialsId: 'ramagopr1',
                        url: "https://github.com/ramagopr1/testrepo.git"
                    )

                   // script {
                     //   env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                     //   env.ROOT_BUILD_CAUSE = java.net.URLEncoder.encode(currentBuild.rawBuild.getCauses()[0].getShortDescription(), "UTF-8")
                   // }

                    // Make sure everythign env-wise is set before this if its used in notifications.
                }
            }
        }



        stage('Build') {
            steps {
                dir("${JOB_BASE_NAME}") {
                    echo "basename"
                    echo "${JOB_BASE_NAME}"
                    echo "/var/jenkins_home/workspace/${JOB_BASE_NAME}/${JOB_BASE_NAME}"
                    sh "/var/jenkins_home/workspace/'${JOB_BASE_NAME}'/'${JOB_BASE_NAME}'/sample.sh"
                }
            }
        }
    }
}