#!groovy

def msg
def artifactId
def additionalArtifactIds
def taskId
def allTaskIds = [] as Set
def compose_url = 'https://kojipkgs.fedoraproject.org/compose/rawhide/latest-Fedora-Rawhide/logs/x86_64/buildinstall-Server-logs/original-pkgsizes.txt'


pipeline {

    agent {
        label 'compose-ci-trigger'                                                                                                                                        
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '14', artifactNumToKeepStr: '100'))
    }

    triggers {
       ciBuildTrigger(
           noSquash: true,
           providerList: [
               rabbitMQSubscriber(
                   name: env.FEDORA_CI_MESSAGE_PROVIDER,
                   overrides: [
                       topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                       queue: 'osci-pipelines-queue-1'
                   ],
                   checks: [
                       [field: '$.artifact.release', expectedValue: '^f34$']
                   ]
               )
           ]
       )
    }

    parameters {
        string(name: 'CI_MESSAGE', defaultValue: '{}', description: 'CI Message')
    }

    stages {
        stage('Trigger Testing') {
            steps {
                script {
                    msg = readJSON text: params.CI_MESSAGE

                    if (msg) {
                        msg['artifact']['builds'].each { build ->
                            def build_id = build['id']
                            def res = sh(script: "./scripts/build_allowed ${compose_url} ${build_id}", returnStatus: true)
                            println("cmd: $it res $res")
                            if (res == 0) {
                                allTaskIds.add(build['task_id'])
                            }
                        }

                        if (allTaskIds) {
                            // compose-ci pipeline can test all given task ids at once, but it is currently
                            // not possible to report only a single result on the whole Fedora update.
                            // Therefore we run the pipeline once, with all given task ids, and we report
                            // only on the first task id (ARTIFACT_ID)
                            // see: https://pagure.io/fedora-ci/general/issue/145

                            taskId = allTaskIds[0]
                            artifactId = "koji-build:${taskId}"
                            // all but first
                            additionalArtifactIds = allTaskIds.findAll{ it != taskId }.collect{ "koji-build:${it}" }.join(',')

                            build(
                                job: 'fedora-ci/compose-ci-pipeline/master',
                                wait: false,
                                parameters: [
                                    string(name: 'ARTIFACT_ID', value: artifactId),
                                    string(name: 'ADDITIONAL_ARTIFACT_IDS', value: additionalArtifactIds)
                                ]
                            )
                        }
                    }
                }
            }
        }
    }
}
