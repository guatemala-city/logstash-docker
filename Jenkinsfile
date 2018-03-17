#!/usr/bin/env groovy
try {
    node('docker') {

        def elastic_docker_registry_host = 'docker.elastic.co'
        def docker_registry_host = env.DOCKER_REGISTRY_HOST ?: 'index.docker.io/v2/'
        def docker_registry_credentials_id = env.DOCKER_REGISTRY_CREDENTIALS_ID?: 'dockerhub_cred'
        def organization = 'guatemalacityex'
        def name = 'logstash'
        def uniqueWorkspace = "build-" +env.BUILD_ID

        def repo_and_tag
        def elastic_repo_and_tag

        def commit_id
        def image
        def version
        def tag

        stage('Checkout') {
            dir(uniqueWorkspace){
                checkout scm

//                commit_id   = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
//                //TODO: move to Python container
//                version         = sh(script: 'python3 ./bin/elastic-version', returnStdout: true).trim()
//
//                if (!env.BRANCH_NAME.toLowerCase().startsWith("master"))
//                    tag = version + '-' + env.BUILD_ID
//                else tag = version
//
//                elastic_repo_and_tag = "${elastic_docker_registry_host}/${name}/${name}:${version}"
//                repo_and_tag = "${organization}/${name}:${tag}"
            }
        }

        withEnv(["DOCKER_REGISTRY_HOST=${docker_registry_host}",
                 "DOCKER_REGISTRY_CREDENTIALS_ID=${docker_registry_credentials_id}",
                 "COMMIT_ID=${commit_id}",
                 "WORKDIR=${uniqueWorkspace}"]) {

            docker.withRegistry("https://${env.DOCKER_REGISTRY_HOST}", env.DOCKER_REGISTRY_CREDENTIALS_ID) {
                def pythonHelper = docker.image('guatemalacityex/python-helper:1.0-logstash')
                def goHelper = docker.image('guatemalacityex/gon-helper:1.0-logstash')

                stage('Clean') {
                    pythonHelper.inside('-u root') {
                        sh "make clean -W clean-demo"
                    }
                }



                stage('Pre-Build') {
                    parallel(
                            'env2yaml': {
                                dir('build/logstash/env2yaml') {
                                    goHelper.inside {
                                        sh "go build"
                                    }
                                }
                            },
                            'dockerfile': {
                                pythonHelper.inside {
                                    sh "make dockerfile -W venv"
                                }
                            },
                            'docker-compose.yml': {
                                pythonHelper.inside {
                                    sh "make docker-compose.yml -W venv"
                                }
                            }
                    )
                }

                stage('Build') {
                    sh 'printenv'



//                image = docker.image(elastic_repo_and_tag)
                }


                stage('Push') {
//                    image.push()
                }

                stage('Promote') {
                    // We can now re-tag and push the 'latest' image.
//                    image.push('latest')
                }
            }

            stage('Cleanup') {
//                sh "docker image rm ${repo_and_tag}"
//                sh "docker image rm ${elastic_repo_and_tag}"
            }

        }
    }
} catch (ex) {
    // If there was an exception thrown, the build failed
    if (currentBuild.result != "ABORTED") {
        // Send e-mail notifications for failed or unstable builds.
        // currentBuild.result must be non-null for this step to work.
        emailext(
                recipientProviders: [
                        [$class: 'DevelopersRecipientProvider'],
                        [$class: 'RequesterRecipientProvider']],
                subject: "Job '${env.JOB_NAME}' - Build ${env.BUILD_DISPLAY_NAME} - FAILED!",
                body: """<p>Job '${env.JOB_NAME}' - Build ${env.BUILD_DISPLAY_NAME} - FAILED:</p>
                        <p>Check console output &QUOT;<a href='${env.BUILD_URL}'>${env.BUILD_DISPLAY_NAME}</a>&QUOT;
                        to view the results.</p>"""
        )
    }

    // Must re-throw exception to propagate error:
    throw ex
}
