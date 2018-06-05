#!groovy

def connectors = [
        'kafka-connect-cassandra':'jcustenborder/kafka-connect-cassandra/master',
        'kafka-connect-cdc-mssql':'jcustenborder/kafka-connect-cdc-mssql/master',
        'kafka-connect-flume-avro':'jcustenborder/kafka-connect-flume-avro/master',
        'kafka-connect-influxdb':'jcustenborder/kafka-connect-influxdb/master',
        'kafka-connect-jms':'jcustenborder/kafka-connect-jms/master',
        'kafka-connect-jmx':'jcustenborder/kafka-connect-jmx/master',
        'kafka-connect-kinesis':'jcustenborder/kafka-connect-kinesis/master',
        'kafka-connect-memcached':'jcustenborder/kafka-connect-memcached/master',
        'kafka-connect-maprdb':'jcustenborder/kafka-connect-maprdb/master',
        'kafka-connect-mqtt':'jcustenborder/kafka-connect-mqtt/master',
        'kafka-connect-rabbitmq':'jcustenborder/kafka-connect-rabbitmq/master',
        'kafka-connect-redis':'jcustenborder/kafka-connect-redis/master',
        'kafka-connect-salesforce':'jcustenborder/kafka-connect-salesforce/master',
        'kafka-connect-simulator':'jcustenborder/kafka-connect-simulator/master',
        'kafka-connect-snmp':'jcustenborder/kafka-connect-snmp/master',
        'kafka-connect-solr':'jcustenborder/kafka-connect-solr/master',
        'kafka-connect-splunk':'jcustenborder/kafka-connect-splunk/master',
        'kafka-connect-spooldir':'jcustenborder/kafka-connect-spooldir/master',
        'kafka-connect-statsd':'jcustenborder/kafka-connect-statsd/master',
        'kafka-connect-syslog':'jcustenborder/kafka-connect-syslog/issue-18',
        'kafka-connect-twitter':'jcustenborder/kafka-connect-twitter/master',
        'kafka-connect-vertica':'jcustenborder/kafka-connect-vertica/master',

        'kafka-connect-transform-archive':'jcustenborder/kafka-connect-transform-archive/master',
        'kafka-connect-transform-cef':'jcustenborder/kafka-connect-transform-cef/master',
        'kafka-connect-transform-common':'jcustenborder/kafka-connect-transform-common/master',
        'kafka-connect-transform-maxmind':'jcustenborder/kafka-connect-transform-maxmind/master',
        'kafka-connect-transform-xml':'jcustenborder/kafka-connect-transform-xml/master',
        
        'kafka-syslog':'jcustenborder/kafka-syslog/initial'
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
