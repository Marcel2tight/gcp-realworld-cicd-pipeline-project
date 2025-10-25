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

        // --- DEPLOYMENT STAGES WITH ANSIBLE ---

        stage('Deploy to DEV') {
            steps {
                echo 'Deploying artifact to the DEV environment...'
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit dev'
            }
            post {
                success {
                    echo '✅ DEV deployment completed successfully!'
                    sh 'ansible dev -a "systemctl is-active JavaWebApp" -u ansadmin'
                }
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
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit stage'
            }
            post {
                success {
                    echo '✅ STAGE deployment completed successfully!'
                    sh 'ansible stage -a "systemctl is-active JavaWebApp" -u ansadmin'
                }
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
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit prod'
            }
            post {
                success {
                    echo '✅ PROD deployment completed successfully!'
                    sh 'ansible prod -a "systemctl is-active JavaWebApp" -u ansadmin'
                }
            }
        }

        // --- ROLLBACK SAFETY NET ---
        stage('Rollback if Needed') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                echo '🚨 Deployment failed! Initiating automatic rollback...'
                sh 'ansible-playbook /etc/ansible/playbooks/rollback-springboot.yml --limit prod'
            }
            post {
                success {
                    echo '✅ Rollback safety check completed'
                    sh 'ansible prod -a "systemctl is-active JavaWebApp" -u ansadmin'
                }
                failure {
                    echo '❌ Rollback failed! Manual intervention required.'
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed for build ${env.BUILD_NUMBER}"
        }
        success {
            echo "🎉 All stages completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed at stage ${env.STAGE_NAME}"
        }
    }
}