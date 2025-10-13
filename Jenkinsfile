pipeline {
    agent any
    tools {
        maven 'M3'  // This should now work
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
                    
                    # Deploy
                    mvn deploy -s settings.xml
                '''
            }
            post {
                success {
                    echo 'Successfully Uploaded Artifact to Nexus Artifactory'
                }
            }
        }
    }
}