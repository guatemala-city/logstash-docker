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
        def version = '5.6.8'
        def tag

        stage('Checkout') {
            dir(uniqueWorkspace){
                checkout scm

                commit_id   = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
//                //TODO: move to Python container
//                version         = sh(script: 'python3 ./bin/elastic-version', returnStdout: true).trim()

                if (!env.BRANCH_NAME.toLowerCase().startsWith("master"))
                    tag = version + '-' + env.BUILD_ID
                else tag = version

                //ORIGINAL_REGISTRY is hardcoded in Python constants
                elastic_repo_and_tag = "${elastic_docker_registry_host}/${name}/${name}:${version}"
                repo_and_tag = "${organization}/${name}:${tag}"
            }
        }

        withEnv(["DOCKER_REGISTRY_HOST=${docker_registry_host}",
                 "DOCKER_REGISTRY_CREDENTIALS_ID=${docker_registry_credentials_id}",
                 "COMMIT_ID=${commit_id}",
                 "WORKDIR=${uniqueWorkspace}"]) {

            docker.withRegistry("https://${env.DOCKER_REGISTRY_HOST}", env.DOCKER_REGISTRY_CREDENTIALS_ID) {
                def pythonHelper = docker.image('guatemalacityex/python-helper:1.0-logstash')
                def goHelper = docker.image('guatemalacityex/go-helper:1.0-logstash')

                stage('Clean') {
                    pythonHelper.inside('-u root') {
                        sh "cd $WORKDIR && make clean -W clean-demo ELASTIC_VERSION=${version}"
                    }
                }



                stage('Pre-Build') {
                    parallel(
                            'env2yaml': {
                                dir("${WORKDIR}/"+'build/logstash/env2yaml') {
                                    goHelper.inside {
                                        sh "go build"
                                    }
                                }
                            },
                            'dockerfile': {
                                pythonHelper.inside {
                                    sh "cd $WORKDIR && make dockerfile -W venv ELASTIC_VERSION=${version}"
                                }
                            },
                            'docker-compose.yml': {
                                pythonHelper.inside {
                                    sh "cd $WORKDIR && make docker-compose.yml -W venv ELASTIC_VERSION=${version}"
                                }
                            }
                    )
                }

                stage('Build') {
                    sh 'printenv'

                    sh "cd $WORKDIR && make build -W venv -W dockerfile -W docker-compose.yml -W env2yaml -W golang ELASTIC_VERSION=${version}"

                    image = docker.image(elastic_repo_and_tag)
                }

                stage('Tests') {
                    try {
                        pythonHelper.inside("-u root -v /var/run/docker.sock:/var/run/docker.sock") {
                            sh "cd $WORKDIR && make test -W venv -W build -W dockerfile -W docker-compose.yml -W env2yaml " +
                                    "-W golang ELASTIC_VERSION=${version}"
                        }
                    } finally {
                        junit "$WORKDIR/tests/reports/*.xml"
                    }
                }


                    stage('Push') {
                        //ORIGINAL_REGISTRY is hardcoded in Python constants - therefore re-tagging required
                        sh "docker image tag ${elastic_repo_and_tag} ${repo_and_tag}"
                        image = docker.image(repo_and_tag)
                        image.push()

                }


            }

            stage('Cleanup') {
                sh "docker image rm ${repo_and_tag}"
                sh "docker image rm ${elastic_repo_and_tag}"
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
