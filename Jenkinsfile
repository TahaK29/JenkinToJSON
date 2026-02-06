pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Check watched.md Changes') {
            steps {
                script {
                    try {
                        // Get list of changed files in this commit
                        def changedFiles = sh(
                            script: '''
                                if git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
                                    git diff --name-only HEAD~1 HEAD
                                else
                                    echo "FIRST_BUILD"
                                fi
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        if (changedFiles == "FIRST_BUILD") {
                            echo "First build detected - checking watched.md"
                            if (fileExists('JenkinToJSON/watched.md')) {
                                sh 'cat JenkinToJSON/watched.md'
                            } else {
                                echo "watched.md not found"
                            }
                        } else if (changedFiles.contains('JenkinToJSON/watched.md')) {
                            echo "watched.md changed - printing contents:"
                            sh 'cat JenkinToJSON/watched.md'
                        } else {
                            echo "watched.md not changed, skipping"
                            currentBuild.result = 'SUCCESS'
                            return
                        }
                    } catch (Exception e) {
                        echo "Error checking changes: ${e.getMessage()}"
                        // Fallback: always print watched.md if check fails
                        if (fileExists('JenkinToJSON/watched.md')) {
                            sh 'cat JenkinToJSON/watched.md'
                        }
                    }
                }
            }
        }
    }
}
