#!groovy
@Library('github.com/cloudogu/ces-build-lib@2.2.1')
import com.cloudogu.ces.cesbuildlib.*

properties([
        // Don't run concurrent builds, because the ITs use the same port causing random failures on concurrent builds.
        disableConcurrentBuilds()
])

node {

    String cesFqdn = findHostName()
    String cesUrl = "https://${cesFqdn}"
    String credentialsId = 'scmCredentials'
    String GIT_COMMIT = ""

    Maven mvn = new MavenWrapper(this)

    catchError {

        stage('Checkout') {
            result = checkout scm
            GIT_COMMIT = result.GIT_COMMIT
        }

        stage('Build') {
            mvn 'clean package -DskipTests'

            archiveArtifacts artifacts: '**/target/*.jar'
        }

        String jacoco = "org.jacoco:jacoco-maven-plugin:0.8.5"

        stage('Test') {
            mvn "${jacoco}:prepare-agent test ${jacoco}:report"
        }

        stage('Integration Test') {
            mvn "${jacoco}:prepare-agent-integration failsafe:integration-test failsafe:verify ${jacoco}:report-integration"
        }

        stage('Static Code Analysis') {
            withCredentials([usernamePassword(credentialsId: 'teamscale-creds', usernameVariable: 'USER', passwordVariable: 'ACCESSKEY')]) {
                script {
                    def teamscaleUrl = env.JENKINS_URL.replace("jenkins", "teamscale")
                    def projectName = currentBuild.fullProjectName.split("/")[1]
                    projectName = projectName.replace("%2F", "-")
                    teamscale includePattern: 'target/site/**/*.xml', credentialsId: 'teamscale-creds', partition: 'Test Coverage', reportFormatId: 'JACOCO', teamscaleProject: projectName, uploadMessage: 'Test Coverage', url: teamscaleUrl, revision: GIT_COMMIT

                    def statusCode = sh returnStatus: true, script:"/home/jenkins/teamscale-buildbreaker --user=$USER --accesskey=$ACCESSKEY --project=${projectName} --server=${teamscaleUrl} --commit=${GIT_COMMIT} --evaluate-findings --fail-on-yellow-findings"
                    if (statusCode == 0) {
                        currentBuild.result = 'SUCCESS';
                        currentBuild.description = 'Teamscale analysis passed successfully';
                    } else if (statusCode == 1) {
                        currentBuild.result = 'FAILURE';
                        currentBuild.description = 'Teamscale analysis detected rule violations';
                    } else if (statusCode == 2) {
                        currentBuild.result = 'FAILURE';
                        currentBuild.description = 'Teamscale analysis detected warnings';
                    } else if (statusCode < 0) {
                        currentBuild.result = 'UNSTABLE';
                        currentBuild.description = 'Could not fetch analysis result from Teamscale (internal error)';
                    } else {
                        currentBuild.result = 'UNSTABLE';
                        currentBuild.description = 'Unknown status code ' + statusCode;
                    }
                }
            }
        }

        stage('Deploy') {
            mvn.useDeploymentRepository([id: cesFqdn, url: "${cesUrl}/nexus", credentialsId: credentialsId, type: 'Nexus3'])

            mvn.deployToNexusRepository('-Dmaven.javadoc.failOnError=false')
        }
    }

    // Archive Unit and integration test results, if any
    junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/TEST-*.xml,**/target/surefire-reports/TEST-*.xml'
}
