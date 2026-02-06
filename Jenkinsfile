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
                        // Get previous build's commit SHA if it exists
                        def previousBuild = currentBuild.getPreviousBuild()
                        def currentCommit = sh(
                            script: 'git rev-parse HEAD',
                            returnStdout: true
                        ).trim()
                        
                        echo "Current commit: ${currentCommit}"
                        
                        // Get list of changed files
                        def changedFiles = ""
                        if (previousBuild == null) {
                            echo "First build detected - checking watched.md"
                            changedFiles = "FIRST_BUILD"
                        } else {
                            // Get previous commit from previous build
                            def previousCommit = sh(
                                script: 'git rev-parse HEAD~1',
                                returnStdout: true
                            ).trim()
                            
                            echo "Previous commit: ${previousCommit}"
                            
                            // Get changed files between commits
                            changedFiles = sh(
                                script: "git diff --name-only ${previousCommit} ${currentCommit}",
                                returnStdout: true
                            ).trim()
                            
                            echo "Changed files: ${changedFiles}"
                        }
                        
                        // Check if watched.md was changed
                        // Also check if this is a manual build (user triggered)
                        def isManualBuild = false
                        try {
                            def causes = currentBuild.rawBuild.getCauses()
                            isManualBuild = causes.any { it instanceof hudson.model.Cause$UserCause }
                        } catch (Exception e) {
                            echo "Could not determine if manual build: ${e.getMessage()}"
                        }
                        
                        if (isManualBuild) {
                            echo "Manual build triggered - will convert watched.md"
                            if (fileExists('watched.md')) {
                                env.SHOULD_CONVERT = 'true'
                            } else {
                                echo "watched.md not found"
                                env.SHOULD_CONVERT = 'false'
                            }
                        } else if (changedFiles == "FIRST_BUILD") {
                            if (fileExists('watched.md')) {
                                echo "First build - will convert watched.md"
                                env.SHOULD_CONVERT = 'true'
                            } else {
                                echo "watched.md not found"
                                env.SHOULD_CONVERT = 'false'
                            }
                        } else if (changedFiles.contains('watched.md')) {
                            echo "watched.md changed - will convert to JSON"
                            env.SHOULD_CONVERT = 'true'
                        } else {
                            echo "watched.md not changed, skipping"
                            echo "Files that changed: ${changedFiles}"
                            env.SHOULD_CONVERT = 'false'
                            currentBuild.result = 'SUCCESS'
                            return
                        }
                    } catch (Exception e) {
                        echo "Error checking changes: ${e.getMessage()}"
                        echo "Stack trace: ${e.getStackTrace().join('\\n')}"
                        // Fallback: always convert watched.md if check fails
                        if (fileExists('watched.md')) {
                            echo "Fallback: converting watched.md due to error"
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
                            cp watched.md temp_content/watched.md
                            
                            echo 'Converting markdown to JSON...'
                            npx markdown-json -s temp_content -d watched.json -p 'watched.md' || {
                                echo 'markdown-json conversion failed, using fallback...'
                                node -e \"
                                    const fs = require('fs');
                                    const mdContent = fs.readFileSync('watched.md', 'utf8');
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
