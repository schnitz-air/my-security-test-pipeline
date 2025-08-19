pipeline {
    // Agent is set to 'none' at the top level because different stages use different environments.
    agent none

    environment {
        // Credentials for both Cortex and Codecov
        CORTEX_API_KEY      = credentials('CORTEX_API_KEY')
        CORTEX_API_KEY_ID   = credentials('CORTEX_API_KEY_ID')
        CORTEX_API_URL      = 'https://api-ms-cxsiamp.xdr.us.paloaltonetworks.com'
        CODECOV_TOKEN       = credentials('codecov-token')
    }

    stages {
        stage('Checkout Source Code') {
            // Use a simple agent to check out the code
            agent any
            steps {
                git url: 'https://github.com/schnitz-air/my-security-test-pipeline.git', branch: 'main'
                
                // Stash the workspace to make it available for the next stage, which runs in a different agent
                stash name: 'source', includes: '**/*'
            }
        }

        stage('Cortex Security Scan') {
            // This stage runs inside a specific Docker container with necessary tools
            agent {
                docker {
                    image 'cimg/node:22.17.0' // An image with curl, jq, git pre-installed
                    args '-u root'
                }
            }
            steps {
                script {
                    // Restore the workspace from the 'Checkout' stage
                    unstash 'source'
                    
                    echo "--- Downloading cortexcli ---"
                    def response = sh(script: """
                        curl --location '${env.CORTEX_API_URL}/public_api/v1/unified-cli/releases/download-link?os=linux&architecture=amd64' \
                             --header 'Authorization: ${env.CORTEX_API_KEY}' \
                             --header 'x-xdr-auth-id: ${env.CORTEX_API_KEY_ID}' \
                             --silent
                    """, returnStdout: true).trim()

                    def downloadUrl = sh(script: "echo '${response}' | jq -r '.signed_url'", returnStdout: true).trim()

                    sh """
                        curl -o cortexcli '${downloadUrl}'
                        chmod +x cortexcli
                        ./cortexcli --version
                    """
                    
                    echo "--- Running Cortex Code Scan ---"
                    // The 'catchError' block ensures the pipeline can continue even if the scan fails
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        sh """
                        ./cortexcli \
                          --api-base-url "${env.CORTEX_API_URL}" \
                          --api-key "${env.CORTEX_API_KEY}" \
                          --api-key-id "${env.CORTEX_API_KEY_ID}" \
                          code scan \
                          --directory "\$(pwd)" \
                          --repo-id 'schnitz-air/my-security-test-pipeline' \
                          --branch '${env.BRANCH_NAME}' \
                          --source "JENKINS" \
                          --create-repo-if-missing
                        """
                    }
                }
            }
        }

        stage('Build, Test, and Simulate') {
            // Use a general-purpose agent for the remaining tasks
            agent any
            steps {
                // Restore the workspace again for this new agent
                unstash 'source'
            }
            stages {
                stage('Install Dependencies') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh '''
                                    echo "--- Creating virtual environment and installing dependencies ---"
                                    python3 -m venv .venv
                                    source .venv/bin/activate
                                    python3 -m pip install --upgrade pip
                                    pip install -r requirements.txt
                                '''
                            }
                        }
                    }
                }
                stage('Run Tests') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh '''
                                    echo "--- Running tests within virtual environment ---"
                                    source .venv/bin/activate
                                    pytest
                                '''
                            }
                        }
                    }
                }
                stage('Test Outgoing Traffic') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'curl -v http://httpbin.org/get'
                                sh 'curl -v https://httpbin.org/get'
                                sh 'curl -v -H "Accept: application/json" http://httpbin.org/get'
                                sh 'curl -v -X POST -H "Content-Type: application/json" -d \'{"key": "value"}\' http://httpbin.org/post'
                                sh 'curl -v --connect-timeout 10 --retry 5 https://httpbin.org/get'
                            }
                        }
                    }
                }
                stage('Network Scan') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'nmap -sT localhost'
                            }
                        }
                    }
                }
                stage('File System Changes') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'touch /tmp/sensitive_file.txt' // Changed to /tmp to avoid permission issues
                                sh 'echo "Suspicious content" > /tmp/sensitive_file.txt'
                                sh 'rm /tmp/sensitive_file.txt'
                            }
                        }
                    }
                }
                stage('Unauthorized Access Attempts') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'for i in {1..5}; do ssh -o BatchMode=yes -o ConnectTimeout=2 invalid-user@localhost; done'
                            }
                        }
                    }
                }
                stage('Simulate Malware') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'curl -O https://secure.eicar.org/eicar.com'
                                sh 'cat eicar.com'
                                sh 'rm eicar.com'
                            }
                        }
                    }
                }
                stage('Privilege Escalation Attempt') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'sudo -n -l || echo "Sudo without password not allowed as expected."'
                            }
                        }
                    }
                }
                stage('Port Scanning') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'nc -zv localhost 1-1000'
                            }
                        }
                    }
                }
                stage('C2 Traffic Simulation') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh 'nc badc2server.com 8080 < /dev/null || echo "C2 connection attempt failed as expected."'
                            }
                        }
                    }
                }
                stage('Upload Coverage to Codecov') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh '''
                                    source .venv/bin/activate
                                    bash <(curl -s https://codecov.io/bash) -t ${CODECOV_TOKEN}
                                '''
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            // Clean up the workspace after the build
            cleanWs()
        }
    }
}
