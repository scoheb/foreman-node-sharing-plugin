node('docker') {
    /* clean out the workspace just to be safe */
    deleteDir()

    /* Grab our source for this build */
    checkout scm

    /* Make sure our directory is there, if Docker creates it, it gets owned by 'root' */
    sh 'mkdir -p $HOME/.m2'

    String containerArgs = '-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.m2:/var/maven/.m2'
    dir('foreman-host-configurator') {
        stage('Build/Test Host Configurator') {
            /* Performing some clever trickery to map our ~/.m2 into the container */

            docker.image('maven:3.3-jdk-8').inside(containerArgs) {
                timestamps {
                    sh 'mvn -B -U -e -Duser.home=/var/maven clean install'
                }
            }
            stage('Archive') {
                junit 'target/surefire-reports/**/*.xml'
                archiveArtifacts artifacts: 'target/**/jms-messaging.hpi', fingerprint: true
                archiveArtifacts artifacts: 'target/diagnostics/**'
            }
        }
    }

    dir('foreman-node-sharing-plugin') {
        stage('Build Plugin') {
            /* Performing some clever trickery to map our ~/.m2 into the container */

            /* Make sure our directory is there, if Docker creates it, it gets owned by 'root' */
            sh 'mkdir -p $HOME/.m2'

            docker.image('maven:3.3-jdk-8').inside(containerArgs) {
                timestamps {
                    sh 'mvn -B -U -e -Dmaven.test.failure.ignore=true -Duser.home=/var/maven clean install -DskipTests'
                }
            }
            def uid = sh(script: 'id -u', returnStdout: true).trim()
            def gid = sh(script: 'id -g', returnStdout: true).trim()

            def buildArgs = "--build-arg=uid=${uid} --build-arg=gid=${gid} src/test/resources/ath-container"
            docker.build('jenkins/ath', buildArgs)

        }

        stage('Test Plugin') {
            docker.image('jenkins/ath').inside(containerArgs) {
                sh '''
                eval $(./vnc.sh 2> /dev/null)
                mvn test -Dmaven.test.failure.ignore=true -Duser.home=/var/maven -Djenkins.version=2.7.3 -DforkCount=1 -B
            '''
            }
        }

        stage('Archive Plugin') {
            junit 'target/surefire-reports/**/*.xml'
            archiveArtifacts artifacts: 'target/**/foreman-*.hpi', fingerprint: true
            archiveArtifacts artifacts: 'target/diagnostics/**'
        }
    }

}