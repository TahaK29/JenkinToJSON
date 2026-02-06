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
                                env.SHOULD_CONVERT = 'true'
                            } else {
                                echo "watched.md not found"
                                env.SHOULD_CONVERT = 'false'
                            }
                        } else if (changedFiles.contains('JenkinToJSON/watched.md')) {
                            echo "watched.md changed - will convert to JSON"
                            env.SHOULD_CONVERT = 'true'
                        } else {
                            echo "watched.md not changed, skipping"
                            env.SHOULD_CONVERT = 'false'
                            currentBuild.result = 'SUCCESS'
                            return
                        }
                    } catch (Exception e) {
                        echo "Error checking changes: ${e.getMessage()}"
                        // Fallback: always convert watched.md if check fails
                        if (fileExists('JenkinToJSON/watched.md')) {
                            env.SHOULD_CONVERT = 'true'
                        } else {
                            env.SHOULD_CONVERT = 'false'
                        }
                    }
                }
            }
        }
        
        stage('Convert MD to JSON') {
            when {
                expression { env.SHOULD_CONVERT == 'true' }
            }
            steps {
                script {
                    echo "Converting watched.md to JSON using markdown-json..."
                    
                    // Use Docker to run Node.js and convert markdown to JSON
                    // This ensures Node.js is available regardless of Jenkins setup
                    sh '''
                        docker run --rm \
                            -v "$(pwd):/workspace" \
                            -w /workspace \
                            node:20-alpine sh -c "
                            echo 'Installing markdown-json package...'
                            npm install markdown-json
                            
                            echo 'Setting up conversion...'
                            mkdir -p temp_content
                            cp JenkinToJSON/watched.md temp_content/watched.md
                            
                            echo 'Converting markdown to JSON...'
                            npx markdown-json -s temp_content -d watched.json -p 'watched.md' || {
                                echo 'markdown-json conversion failed, using fallback...'
                                node -e \"
                                    const fs = require('fs');
                                    const mdContent = fs.readFileSync('JenkinToJSON/watched.md', 'utf8');
                                    const jsonOutput = JSON.stringify({ 
                                        filename: 'watched.md',
                                        content: mdContent,
                                        timestamp: new Date().toISOString()
                                    }, null, 2);
                                    fs.writeFileSync('watched.json', jsonOutput);
                                \"
                            }
                            
                            echo '=== JSON Output ==='
                            if [ -f watched.json ]; then
                                cat watched.json
                            else
                                echo 'Error: JSON file was not created'
                                exit 1
                            fi
                            
                            # Cleanup
                            rm -rf temp_content node_modules package-lock.json
                        "
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed"
            // Cleanup temp files
            sh 'rm -rf temp_content watched.json || true'
        }
    }
}
