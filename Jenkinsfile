pipeline {
    agent any
    
    environment {
        SLACK_CHANNEL = '#deployments'
    }
    
    stages {
        stage('Pipeline Started') {
            steps {
                withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                    sh """
                    curl -s -X POST -H 'Content-type: application/json' \
                    --data '{"channel":"${env.SLACK_CHANNEL}","text":"🚀 DEPLOYMENT PIPELINE STARTED\\n*Application:* JavaWebApp\\n*Build:* #${env.BUILD_NUMBER}\\n*Branch:* ${env.GIT_BRANCH}\\n*Initiator:* ${env.USER_ID}"}' \
                    ${SLACK_WEBHOOK}
                    """
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
                sh """mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=java-webapp \
                        -Dsonar.host.url=http://10.128.0.6:9000 \
                        -Dsonar.login=sqp_73f4f6c60a9cb39ce13bc2c8272dc83a313a904b"""
            }
        }
        
        stage("Upload Artifact to Nexus") {
            steps {
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
            
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"channel":"${env.SLACK_CHANNEL}","text":"🚀 ${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}\\n🔗 ${env.BUILD_URL}"}' \
                ${SLACK_WEBHOOK}
                """
            }
        }
        
        success {
            echo "🎉 All stages completed successfully!"
            
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"channel":"${env.SLACK_CHANNEL}","text":"✅ DEPLOYMENT SUCCESS!\\n*Application:* JavaWebApp\\n*Build:* #${env.BUILD_NUMBER}\\n*Environments:* ✅ Dev → ✅ Stage → ✅ Prod\\n*Time:* \$(date)\\n*URL:* ${env.BUILD_URL}"}' \
                ${SLACK_WEBHOOK}
                """
            }
        }
        
        failure {
            echo "❌ Pipeline failed at stage ${env.STAGE_NAME}"
            
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"channel":"${env.SLACK_CHANNEL}","text":"❌ DEPLOYMENT FAILED!\\n*Build:* #${env.BUILD_NUMBER}\\n*Application:* JavaWebApp\\n*Failed Stage:* ${env.STAGE_NAME}\\n*URL:* ${env.BUILD_URL}\\n*Time:* \$(date)"}' \
                ${SLACK_WEBHOOK}
                """
            }
        }
        
        unstable {
            echo "⚠️ Pipeline is unstable"
            
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                curl -s -X POST -H 'Content-type: application/json' \
                --data '{"channel":"${env.SLACK_CHANNEL}","text":"⚠️ BUILD UNSTABLE\\n*Build:* #${env.BUILD_NUMBER}\\n*Application:* JavaWebApp\\n*URL:* ${env.BUILD_URL}"}' \
                ${SLACK_WEBHOOK}
                """
            }
        }
    }
}