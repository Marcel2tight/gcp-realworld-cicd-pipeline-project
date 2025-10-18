pipeline {
    agent any
    stages {
        stage('Validate Project') {
            steps {
                sh 'mvn validate'
            }
        }
        stage('Unit Test'){
            steps {
                sh 'mvn test'
            }
        }
        stage('Integration Test'){
            steps {
                // Ensure unit tests are skipped as they ran in the previous stage
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage('App Packaging'){
            steps {
                sh 'mvn package'
            }
        }
        stage ('Checkstyle Code Analysis'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage('SonarQube Inspection') {
            steps {
                sh """mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=java-webapp \
                        -Dsonar.host.url=http://10.128.0.7:9000 \
                        -Dsonar.login=sqp_39f4a576f2be6657a3ddad65a036e82e0ec20dea"""
            }
        }
        stage("Upload Artifact to Nexus"){
            steps{
                sh 'mvn deploy'
            }
            post {
                success {
                    echo 'Successfully Uploaded Artifact to Nexus Artifactory'
                }
            }
        }

        // --- NEW DEPLOYMENT STAGES BELOW ---

        stage('Deploy to DEV') {
            steps {
                echo 'Deploying artifact to the DEV environment...'
                // Placeholder for your actual deployment script (e.g., Ansible, shell, custom script)
                // sh './deploy-dev.sh'
            }
        }

        stage('Approval for STAGE Deployment') {
            steps {
                // Wait for manual approval before proceeding to the Stage environment
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Promote to STAGE environment? Requires QA/Tester approval.',
                          submitter: 'testers,qa-team' // Specify groups or users allowed to approve
                }
            }
        }

        stage('Deploy to STAGE') {
            steps {
                echo 'Deploying artifact to the STAGE environment...'
                // Placeholder for your actual deployment script
                // sh './deploy-stage.sh'
            }
        }

        stage('Approval for PROD Deployment') {
            steps {
                // Wait for manual approval before proceeding to the Production environment
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Final approval: Promote to PRODUCTION environment? Requires Management/Release Team approval.',
                          submitter: 'managers,release-team' // Specify groups or users allowed to approve
                }
            }
        }

        stage('Deploy to PROD') {
            steps {
                echo 'Deploying artifact to the PRODUCTION environment... 🚀'
                // Placeholder for your actual deployment script
                // sh './deploy-prod.sh'
            }
            post {
                success {
                    echo ' Deployment to PROD completed successfully!'
                }
                failure {
                    echo ' PROD deployment failed! Immediate review required.'
                }
            }
        }
    }
}