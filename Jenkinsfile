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
        
        // SECURE SONARQUBE INSPECTION
        stage('SonarQube Inspection') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh """mvn clean verify sonar:sonar \\
                        -Dsonar.projectKey=java-webapp \\
                        -Dsonar.host.url=http://10.128.0.3:9000 \\
                        -Dsonar.login=\${SONAR_TOKEN}"""
                }
            }
        }
        
        // SECURE NEXUS DEPLOYMENT - UPDATED
        stage("Upload Artifact To Nexus"){
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexusdeploymentrepo', 
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    // Use clean deploy instead of package deploy
                    sh 'mvn clean deploy'
                }
            }
            post {
                success {
                    echo 'Successfully Uploaded Artifact to Nexus Artifactory'
                }
            }
        }
    }
}