#!groovy

def connectors = [
        'siem-proxy':'jcustenborder/siem-proxy/initial'
]

properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '30')),
    disableConcurrentBuilds(),
    pipelineTriggers([
        upstream(threshold: 'SUCCESS', upstreamProjects: connectors.values().join(','))
    ])
])

node {
    deleteDir()
    checkout scm

    def root_path = 'output'
    def rpm_path = "${root_path}/yum"
    def deb_path = "${root_path}/deb/"

    sh "mkdir -p ${rpm_path} ${deb_path}"

    stage('copy') {
        connectors.each { name, projectName ->
            sh "echo Copying artifacts for ${projectName}"
            copyArtifacts(
                filter: "target/${name}-*.rpm",
                flatten: true,
                projectName: "${projectName}",
                selector: lastSuccessful(),
                target: "${rpm_path}"
            )
            copyArtifacts(
                    filter: "target/${name}-*.deb",
                    flatten: true,
                    projectName: "${projectName}",
                    selector: lastSuccessful(),
                    target: "${deb_path}"
            )
        }
    }

    stage('repo') {
        docker.image('jcustenborder/packaging-centos-7:45').inside {
            sh "createrepo ${rpm_path}"
        }
    }


    stage('publish') {
        sshagent (credentials: ['eafaa2d0-dc8a-4bdc-9f0b-f6d290c9a6b5']) {
            withCredentials([string(credentialsId: 'package_ssh_hostname', variable: 'hostname')]) {
                sh "rsync -e 'ssh -o StrictHostKeyChecking=no' -avz --delete '${root_path}/' '${hostname}:/mnt/packages/jcustenborder/'"
            }
        }
    }

    stage('deploy') {
        sshagent (credentials: ['eafaa2d0-dc8a-4bdc-9f0b-f6d290c9a6b5']) {
            def commands=[
                    'yum clean all',
                    "yum '--disablerepo=*' --enablerepo=jcustenborder -y -q -e 0 install '*'",
                    "yum '--disablerepo=*' --enablerepo=jcustenborder -y -q -e 0 upgrade '*'",
                    "systemctl restart connect-distributed",
                    "systemctl restart connect-standalone"
            ]

            parallel 'connect-01': {
                commands.each { command ->
                    sh "ssh -o StrictHostKeyChecking=no root@connect-01 \"${command}\""
                }
            }, 'connect-02': {
                commands.each { command ->
                    sh "ssh -o StrictHostKeyChecking=no root@connect-02 \"${command}\""
                }
            }
        }
    }

}
