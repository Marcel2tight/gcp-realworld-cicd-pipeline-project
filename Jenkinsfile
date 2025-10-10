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
                sh  """mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=java-webapp \
                        -Dsonar.host.url=http://10.128.0.3:9000 \
                        -Dsonar.login=sqp_a7c60123f30fdb83e8398b76accf61c24d331c29"""
            }
        }
        stage("Upload Artifact To Nexus"){
            steps{
                 sh 'mvn deploy'
            }
            post {
                success {
                  echo 'Successfully Uploaded Artifact to Nexus Artifactory'
                }
            }
        }
    }
}
