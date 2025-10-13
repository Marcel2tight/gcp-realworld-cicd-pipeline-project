pipeline {
    agent any
    tools {
        maven 'M3'
    }
    stages {
        stage("Build") {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage("Test Nexus Connection") {
            steps {
                sh '''
                    echo "Testing basic Nexus connectivity..."
                    if curl -s -u admin:adminadmin http://10.128.0.7:8081/ > /dev/null; then
                        echo "✓ Nexus is reachable"
                    else
                        echo "✗ Cannot reach Nexus"
                        exit 1
                    fi
                    
                    echo "Testing repository access..."
                    if curl -s -u admin:adminadmin http://10.128.0.7:8081/repository/java-webapp-snapshots/ > /dev/null; then
                        echo "✓ Repository is accessible"
                    else
                        echo "✗ Cannot access repository - check permissions"
                        exit 1
                    fi
                '''
            }
        }
        
        stage("Upload Artifact To Nexus") {
            steps {
                sh '''
                    # Create settings.xml with credentials
                    cat > settings.xml << 'EOF'
                    <?xml version="1.0" encoding="UTF-8"?>
                    <settings>
                      <servers>
                        <server>
                          <id>nexusdeploymentrepo</id>
                          <username>admin</username>
                          <password>adminadmin</password>
                        </server>
                      </servers>
                    </settings>
                    EOF
                    
                    # Deploy with detailed logging
                    mvn -B deploy -s settings.xml -X
                '''
            }
            post {
                success {
                    echo 'Successfully Uploaded Artifact to Nexus Artifactory'
                }
                failure {
                    echo 'Failed to upload artifact - check Nexus logs and permissions'
                }
            }
        }
    }
}