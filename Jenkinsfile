pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#deployments'
        NEXUS_USERNAME = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
    }
    
    stages {
        stage('Pipeline Started') {
            steps {
                withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                    script {
                        def message = """
🚀 DEPLOYMENT PIPELINE STARTED
*Application:* JavaWebApp
*Build:* #${env.BUILD_NUMBER}
*Branch:* ${env.GIT_BRANCH}
*Initiator:* ${env.USER_ID}
"""
                        def webhook = env.SLACK_WEBHOOK_URL
                        sh """
                            curl -s -X POST -H "Content-type: application/json" \
                            --data '{"channel":"${env.SLACK_CHANNEL}", "text":"${message}"}' \
                            '${webhook}'
                        """
                    }
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
                        withSonarQubeEnv('sonarqube') {
                            sh """
                            mvn clean verify sonar:sonar \
                                -Dsonar.projectKey=java-webapp
                            """
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                script {
                    echo "📊 SonarQube analysis submitted"
                    echo "🔗 Dashboard: http://10.128.0.7:9000/dashboard?id=java-webapp"
                    
                    // Try automated quality gate first
                    def qualityGatePassed = false
                    def maxAttempts = 3
                    def attempt = 1
                    
                    while (attempt <= maxAttempts && !qualityGatePassed) {
                        echo "🔄 Quality Gate check attempt ${attempt}/${maxAttempts}"
                        
                        try {
                            timeout(time: 5, unit: 'MINUTES') {
                                waitForQualityGate abortPipeline: true
                            }
                            qualityGatePassed = true
                            echo "✅ Quality Gate passed on attempt ${attempt}!"
                        } catch (Exception e) {
                            echo "⚠️ Quality Gate attempt ${attempt} failed: ${e.message}"
                            if (attempt < maxAttempts) {
                                echo "⏳ Waiting 60 seconds before retry..."
                                sleep time: 60, unit: 'SECONDS'
                            }
                            attempt++
                        }
                    }
                    
                    if (!qualityGatePassed) {
                        echo "🚨 Automated Quality Gate failed after ${maxAttempts} attempts"
                        echo "📋 Manual review required at: http://10.128.0.7:9000/dashboard?id=java-webapp"
                        
                        // Manual approval for quality gate
                        timeout(time: 5, unit: 'MINUTES') {
                            input(
                                message: "SonarQube Quality Gate is delayed. Manually verify results at http://10.128.0.7:9000/dashboard?id=java-webapp and proceed?",
                                ok: "Quality Verified - Proceed",
                                submitterParameter: 'qualityApprover'
                            )
                        }
                        echo "✅ Manual Quality Gate approval received from ${params.qualityApprover}"
                    }
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
                script {
                    def message = """
🔔 ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}
🔗 ${env.BUILD_URL}
📊 SonarQube: http://10.128.0.7:9000/dashboard?id=java-webapp
"""
                    def webhook = env.SLACK_WEBHOOK_URL
                    sh """
                        curl -s -X POST -H "Content-type: application/json" \
                        --data '{"channel":"${env.SLACK_CHANNEL}", "text":"${message}"}' \
                        '${webhook}'
                    """
                }
            }
        }
        
        success {
            echo "🎉 All stages completed successfully!"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                script {
                    def message = """
✅ DEPLOYMENT SUCCESS!
*Application:* JavaWebApp
*Build:* #${env.BUILD_NUMBER}
*Environments:* ✅ Dev → ✅ Stage → ✅ Prod
*SonarQube:* http://10.128.0.7:9000/dashboard?id=java-webapp
*URL:* ${env.BUILD_URL}
"""
                    def webhook = env.SLACK_WEBHOOK_URL
                    sh """
                        curl -s -X POST -H "Content-type: application/json" \
                        --data '{"channel":"${env.SLACK_CHANNEL}", "text":"${message}"}' \
                        '${webhook}'
                    """
                }
            }
        }
        
        failure {
            echo "❌ Pipeline failed at stage ${env.STAGE_NAME}"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                script {
                    def message = """
❌ DEPLOYMENT FAILED!
*Build:* #${env.BUILD_NUMBER}
*Application:* JavaWebApp
*Failed Stage:* ${env.STAGE_NAME}
*URL:* ${env.BUILD_URL}
"""
                    def webhook = env.SLACK_WEBHOOK_URL
                    sh """
                        curl -s -X POST -H "Content-type: application/json" \
                        --data '{"channel":"${env.SLACK_CHANNEL}", "text":"${message}"}' \
                        '${webhook}'
                    """
                }
            }
        }
        
        unstable {
            echo "⚠️ Pipeline is unstable"
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK_URL')]) {
                script {
                    def message = """
⚠️ BUILD UNSTABLE
*Build:* #${env.BUILD_NUMBER}
*Application:* JavaWebApp
*URL:* ${env.BUILD_URL}
"""
                    def webhook = env.SLACK_WEBHOOK_URL
                    sh """
                        curl -s -X POST -H "Content-type: application/json" \
                        --data '{"channel":"${env.SLACK_CHANNEL}", "text":"${message}"}' \
                        '${webhook}'
                    """
                }
            }
        }
    }
}