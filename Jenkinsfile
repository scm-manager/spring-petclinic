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

    Maven mvn = new MavenWrapper(this)

    catchError {

        stage('Checkout') {
            checkout scm
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

            def sonarQube = new SonarQube(this, [usernamePassword: credentialsId, sonarHostUrl: "${cesUrl}/sonar"])

            sonarQube.analyzeWith(mvn)
        }

        stage('Deploy') {
            mvn.useDeploymentRepository([id: cesFqdn, url: "${cesUrl}/nexus", credentialsId: credentialsId, type: 'Nexus3'])

            mvn.deployToNexusRepository('-Dmaven.javadoc.failOnError=false')
        }
    }

    // Archive Unit and integration test results, if any
    junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/TEST-*.xml,**/target/surefire-reports/TEST-*.xml'
}
