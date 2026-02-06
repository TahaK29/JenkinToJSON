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
                    // No Docker: Jenkins runs inside a container that has no Docker CLI.
                    // Use standalone Node.js binary or fallback to simple JSON.
                    sh '''
                        set -e
                        if command -v node >/dev/null 2>&1; then
                            echo "Using system Node.js: $(node --version)"
                            NODE_CMD=node
                            NPM_CMD=npm
                            NPX_CMD=npx
                        else
                            echo "Node.js not found. Downloading standalone Node.js..."
                            ARCH=$(uname -m)
                            case "$ARCH" in
                                x86_64) NODE_ARCH="x64" ;;
                                aarch64|arm64) NODE_ARCH="arm64" ;;
                                *) echo "Unsupported arch: $ARCH"; exit 1 ;;
                            esac
                            NODE_VERSION="v20.18.0"
                            NODE_DIR="node-${NODE_VERSION}-linux-${NODE_ARCH}"
                            NODE_TGZ="${NODE_DIR}.tar.gz"
                            NODE_URL="https://nodejs.org/dist/${NODE_VERSION}/${NODE_TGZ}"
                            if command -v curl >/dev/null 2>&1; then
                                curl -fsSL "$NODE_URL" -o "$NODE_TGZ"
                            else
                                wget -q "$NODE_URL" -O "$NODE_TGZ"
                            fi
                            tar -xzf "$NODE_TGZ"
                            export PATH="$(pwd)/${NODE_DIR}/bin:$PATH"
                            NODE_CMD=node
                            NPM_CMD=npm
                            NPX_CMD=npx
                            echo "Using downloaded Node.js: $($NODE_CMD --version)"
                        fi

                        echo "Installing markdown-json package..."
                        $NPM_CMD install markdown-json

                        echo "Setting up conversion..."
                        mkdir -p temp_content
                        cp watched.md temp_content/watched.md

                        echo "Converting markdown to JSON..."
                        # markdown-json -d expects a directory, creates output.json inside it
                        mkdir -p output_dir
                        if $NPX_CMD markdown-json -s temp_content -d output_dir -p "watched.md" 2>&1; then
                            # Check for output file (markdown-json creates output.json in the dist directory)
                            if [ -f "output_dir/output.json" ]; then
                                cp output_dir/output.json watched.json
                                echo "markdown-json conversion successful"
                            else
                                echo "markdown-json ran but output.json not found, using fallback..."
                                $NODE_CMD -e "
                                    const fs = require('fs');
                                    const mdContent = fs.readFileSync('watched.md', 'utf8');
                                    const jsonOutput = JSON.stringify({
                                        filename: 'watched.md',
                                        content: mdContent,
                                        timestamp: new Date().toISOString()
                                    }, null, 2);
                                    fs.writeFileSync('watched.json', jsonOutput);
                                "
                            fi
                        else
                            echo "markdown-json conversion failed, using fallback..."
                            $NODE_CMD -e "
                                const fs = require('fs');
                                const mdContent = fs.readFileSync('watched.md', 'utf8');
                                const jsonOutput = JSON.stringify({
                                    filename: 'watched.md',
                                    content: mdContent,
                                    timestamp: new Date().toISOString()
                                }, null, 2);
                                fs.writeFileSync('watched.json', jsonOutput);
                            "
                        fi

                        echo "=== JSON Output ==="
                        if [ -f watched.json ]; then
                            cat watched.json
                        else
                            echo "Error: JSON file was not created"
                            exit 1
                        fi

                        rm -rf temp_content output_dir node_modules package-lock.json node-*.tar.gz node-v*
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
