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
        }

        // --- DEPLOYMENT STAGES ---
        stage('Deploy to DEV') {
            steps {
                echo 'Deploying artifact to the DEV environment...'
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit dev --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
        }

        stage('Approval for STAGE Deployment') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Promote to STAGE environment?',
                          submitter: 'testers,qa-team'
                }
            }
        }

        stage('Deploy to STAGE') {
            steps {
                echo 'Deploying artifact to the STAGE environment...'
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit stage --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
        }

        stage('Approval for PROD Deployment') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Final approval: Promote to PRODUCTION?',
                          submitter: 'managers,release-team'
                }
            }
        }

        stage('Deploy to PROD') {
            steps {
                echo 'Deploying artifact to the PRODUCTION environment... 🚀'
                sh 'ansible-playbook /etc/ansible/playbooks/deploy-springboot.yml --limit prod --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
        }

        stage('Rollback if Needed') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                echo '🚨 Deployment failed! Initiating automatic rollback...'
                sh 'ansible-playbook /etc/ansible/playbooks/rollback-springboot.yml --limit prod --private-key=/var/lib/jenkins/.ssh/ansible_key'
            }
        }
    }
    
    post {
        always {
            echo "Pipeline execution completed for build ${env.BUILD_NUMBER}"
            
            // Direct Slack notification
            sh """
            curl -s -X POST -H 'Content-type: application/json' \
            --data '{"text":"🚀 ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}\\n🔗 ${env.BUILD_URL}"}' \
            https://hooks.slack.com/services/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
            """
        }
        
        success {
            echo "🎉 All stages completed successfully!"
            
            // Success notification
            sh """
            curl -s -X POST -H 'Content-type: application/json' \
            --data '{"text":"✅ DEPLOYMENT SUCCESS!\\nApplication: JavaWebApp\\nBuild: #${env.BUILD_NUMBER}\\nEnvironments: ✅ Dev → ✅ Stage → ✅ Prod\\nTime: $(date)"}' \
            https://hooks.slack.com/services/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
            """
        }
        
        failure {
            echo "❌ Pipeline failed at stage ${env.STAGE_NAME}"
            
            // Failure notification
            sh """
            curl -s -X POST -H 'Content-type: application/json' \
            --data '{"text":"❌ DEPLOYMENT FAILED!\\nBuild: #${env.BUILD_NUMBER}\\nApplication: JavaWebApp\\nFailed Stage: ${env.STAGE_NAME}\\nURL: ${env.BUILD_URL}"}' \
            https://hooks.slack.com/services/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
            """
        }
    }
}