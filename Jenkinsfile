#!groovy

def msg
def artifactId
def allTaskIds = [] as Set
def compose_url = 'https://kojipkgs.fedoraproject.org/compose/rawhide/latest-Fedora-Rawhide/logs/x86_64/buildinstall-Server-logs/original-pkgsizes.txt'


pipeline {

    agent {
        label 'compose-ci-trigger'                                                                                                                                        
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        skipDefaultCheckout()
    }

    triggers {
       ciBuildTrigger(
           noSquash: true,
           providerList: [
               rabbitMQSubscriber(
                   name: env.FEDORA_CI_MESSAGE_PROVIDER,
                   overrides: [
                       topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                       queue: 'osci-pipelines-queue-10'
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
                    msg = readJSON text: CI_MESSAGE

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
                            allTaskIds.each { taskId ->
                                artifactId = "koji-build:${taskId}"

                                build(
                                    job: 'fedora-ci/compose-ci-pipeline/master',
                                    wait: false,
                                    parameters: [
                                        string(name: 'ARTIFACT_ID', value: artifactId)
                                    ]
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}
