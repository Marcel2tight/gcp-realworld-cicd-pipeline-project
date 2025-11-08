pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#deployments'
    }
    
    stages {
        stage('Pipeline Started') {
            steps {
                withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                    sh '''
                        curl -s -X POST -H "Content-type: application/json" \
                        --data "{\"channel\":\"'${env.SLACK_CHANNEL}'\",\"text\":\"🚀 DEPLOYMENT PIPELINE STARTED\\n*Application:* JavaWebApp\\n*Build:* #'${env.BUILD_NUMBER}'\\n*Branch:* '${env.GIT_BRANCH}'\\n*Initiator:* '${env.USER_ID}'\"}" \
                        ${SLACK_WEBHOOK_URL}
                    '''
                }
            }
        }
        
        stage('Validate Project') {
            steps {
                sh 'mvn validate'
            }
        }
        
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Integration Test') {
            steps {
                // Ensure unit tests are skipped as they ran in the previous stage
                sh 'mvn verify -DskipUnitTests'
            }
        }
        
        stage('App Packaging') {
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }
        
        stage('SonarQube Inspection') {
            steps {
                retry(2) {
                    timeout(time: 15, unit: 'MINUTES') {
                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                            mvn clean verify sonar:sonar \
                                -Dsonar.projectKey=java-webapp \
                                -Dsonar.host.url=http://10.128.0.7:9000 \
                                -Dsonar.login=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage("Upload Artifact to Nexus") {
            steps {
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
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit dev --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
            post {
                success {
                    echo '✅ DEV deployment completed successfully!'
                    sh 'ansible dev -a "systemctl is-active JavaWebApp" -u ansadmin --private-key=/var/lib/jenkins/.ssh/ansible_key'
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
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit stage --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
            post {
                success {
                    echo '✅ STAGE deployment completed successfully!'
                    sh 'ansible stage -a "systemctl is-active JavaWebApp" -u ansadmin --private-key=/var/lib/jenkins/.ssh/ansible_key'
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
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit prod --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
            post {
                success {
                    echo '✅ PROD deployment completed successfully!'
                    sh 'ansible prod -a "systemctl is-active JavaWebApp" -u ansadmin --private-key=/var/lib/jenkins/.ssh/ansible_key'
                }
            }
        }

        // --- ROLLBACK SAFETY NET ---
        stage('Rollback if Needed') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                echo '🔄 Deployment failed! Initiating automatic rollback...'
                sh 'ansible-playbook /etc/ansible/playbooks/rollback-springboot.yml --limit prod --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
            post {
                success {
                    echo '✅ Rollback safety check completed'
                    sh 'ansible prod -a "systemctl is-active JavaWebApp" -u ansadmin --private-key=/var/lib/jenkins/.ssh/ansible_key'
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
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                sh '''
                    curl -s -X POST -H "Content-type: application/json" \
                    --data "{\"channel\":\"'${env.SLACK_CHANNEL}'\",\"text\":\"🔔 '${env.JOB_NAME}' - Build #'${env.BUILD_NUMBER}' - '${currentBuild.currentResult}'\\n🔗 '${env.BUILD_URL}'\"}" \
                    ${SLACK_WEBHOOK_URL}
                '''
            }
        }
        
        success {
            echo "🎉 All stages completed successfully!"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                sh '''
                    curl -s -X POST -H "Content-type: application/json" \
                    --data "{\"channel\":\"'${env.SLACK_CHANNEL}'\",\"text\":\"✅ DEPLOYMENT SUCCESS!\\n*Application:* JavaWebApp\\n*Build:* #'${env.BUILD_NUMBER}'\\n*Environments:* ✅ Dev → ✅ Stage → ✅ Prod\\n*Time:* \$(date)\\n*URL:* '${env.BUILD_URL}'\"}" \
                    ${SLACK_WEBHOOK_URL}
                '''
            }
        }
        
        failure {
            echo "❌ Pipeline failed at stage ${env.STAGE_NAME}"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                sh '''
                    curl -s -X POST -H "Content-type: application/json" \
                    --data "{\"channel\":\"'${env.SLACK_CHANNEL}'\",\"text\":\"❌ DEPLOYMENT FAILED!\\n*Build:* #'${env.BUILD_NUMBER}'\\n*Application:* JavaWebApp\\n*Failed Stage:* '${env.STAGE_NAME}'\\n*URL:* '${env.BUILD_URL}'\\n*Time:* \$(date)\"}" \
                    ${SLACK_WEBHOOK_URL}
                '''
            }
        }
        
        unstable {
            echo "⚠️ Pipeline is unstable"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                sh '''
                    curl -s -X POST -H "Content-type: application/json" \
                    --data "{\"channel\":\"'${env.SLACK_CHANNEL}'\",\"text\":\"⚠️ BUILD UNSTABLE\\n*Build:* #'${env.BUILD_NUMBER}'\\n*Application:* JavaWebApp\\n*URL:* '${env.BUILD_URL}'\"}" \
                    ${SLACK_WEBHOOK_URL}
                '''
            }
        }
    }
}