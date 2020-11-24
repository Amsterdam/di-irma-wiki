#!groovy
def tryStep(String message, Closure block, Closure tearDown = null) {
    try {
        block();
    }
    catch (Throwable t) {
        slackSend message: "${env.JOB_NAME}: ${message} failure ${env.BUILD_URL}", channel: '#ci-channel', color: 'danger'
        throw t;
    }
    finally {
        if (tearDown) {
            tearDown();
        }
    }
}
node {
    stage("Checkout") {
        checkout scm
    }
    stage("Build image - wiki") {
        tryStep "build", {
            def dockerfile = './Dockerfile.wiki'
            def image = docker.build("docker-registry.secure.amsterdam.nl/irma/irma-wiki:${env.BUILD_NUMBER}", "-f ${dockerfile} ./")
            image.push()
        }
    }
    stage("Build image - db") {
        tryStep "build", {
            def dockerfile = './Dockerfile.db'
            def image = docker.build("docker-registry.secure.amsterdam.nl/irma/irma-wiki-db:${env.BUILD_NUMBER}", "-f ${dockerfile} ./")
            image.push()
        }
    }
}

String BRANCH = "${env.BRANCH_NAME}"
if (BRANCH == "master" || BRANCH == "develop") {
    node {
        stage('Push acceptance image - wiki') {
            tryStep "image tagging", {
                def image = docker.image("docker-registry.secure.amsterdam.nl/irma/irma-wiki:${env.BUILD_NUMBER}")
                image.pull()
                image.push("acceptance")
            }
        }
        stage('Push acceptance image - db') {
            tryStep "image tagging", {
                def image = docker.image("docker-registry.secure.amsterdam.nl/irma/irma-wiki-db:${env.BUILD_NUMBER}")
                image.pull()
                image.push("acceptance")
            }
        }
    }
    node {
        stage("Deploy to ACC") {
            tryStep "deployment", {
                build job: 'Subtask_Openstack_Playbook',
                    parameters: [
                            [$class: 'StringParameterValue', name: 'INVENTORY', value: 'acceptance'],
                            [$class: 'StringParameterValue', name: 'PLAYBOOK', value: 'deploy.yml'],
                            [$class: 'StringParameterValue', name: 'PLAYBOOKPARAMS', value: "-e cmdb_id=app_irma-wiki"]
                    ]
            }
        }
    }
if (BRANCH == "master") {
    // stage('Waiting for approval') {
    //     slackSend channel: '#ci-channel', color: 'warning', message: 'irma_frontend is waiting for Production Release - please confirm'
    //     input "Deploy to Production?"
    // }
    node {
        stage('Push production image - wiki') {
        tryStep "image tagging", {
            def image = docker.image("docker-registry.secure.amsterdam.nl/irma/irma-wiki:${env.BUILD_NUMBER}")
                image.pull()
                image.push("production")
                image.push("latest")
            }
        }
        stage('Push production image - db') {
        tryStep "image tagging", {
            def image = docker.image("docker-registry.secure.amsterdam.nl/irma/irma-wiki-db:${env.BUILD_NUMBER}")
                image.pull()
                image.push("production")
                image.push("latest")
            }
        }
    }
    node {
        stage("Deploy") {
            tryStep "deployment", {
                build job: 'Subtask_Openstack_Playbook',
                parameters: [
                    [$class: 'StringParameterValue', name: 'INVENTORY', value: 'production'],
                    [$class: 'StringParameterValue', name: 'PLAYBOOK', value: 'deploy.yml'],
                    [$class: 'StringParameterValue', name: 'PLAYBOOKPARAMS', value: "-e cmdb_id=app_irma-wiki"]
                ]
            }
        }
    }
  }
}
