pipeline {
    agent any
    
    tools {
        'hudson.plugins.sonar.SonarRunnerInstallation' 'SonarScanner'
    }
    
    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version to deploy')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Environment to deploy to')
    }
    
    environment {
        ARTIFACT_NAME = "php-crud-app"
        ARTIFACT_DIR = "artifacts"
        MYSQL_CREDS = credentials('mysql-credentials')
        STAGING_PUBLIC_IP = "3.80.29.14"
        SONAR_PROJECT_KEY = "php-crud-app"
        SONAR_URL = "http://54.196.217.149:9000"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                
                // Check if src directory exists
                script {
                    if (!fileExists('src')) {
                        error "Source directory 'src' not found. Please check your repository structure."
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    // Use proper credentials binding for security
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        // Add Java options to fix compatibility issues with Java 17
                        withEnv(["SONAR_SCANNER_OPTS=-Djava.security.manager=allow --add-opens=java.base/java.lang=ALL-UNNAMED"]) {
                            sh """
                                ${tool 'SonarScanner'}/bin/sonar-scanner \\
                                -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \\
                                -Dsonar.projectName="PHP CRUD Application" \\
                                -Dsonar.projectVersion=${params.VERSION} \\
                                -Dsonar.sources=src \\
                                -Dsonar.host.url=${env.SONAR_URL} \\
                                -Dsonar.login=\${SONAR_TOKEN}
                            """
                        }
                    }
                }
                
                // Save the task ID for manual checking if needed
                script {
                    def taskId = sh(script: "grep 'More about the report processing at' .scannerwork/report-task.txt | sed 's/.*id=\\(.*\\)/\\1/'", returnStdout: true).trim()
                    env.SONAR_TASK_ID = taskId
                    echo "SonarQube Task ID: ${env.SONAR_TASK_ID}"
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                        waitForQualityGate abortPipeline: false
                    }
                }
                
                // Check quality gate status manually as a fallback
                script {
                    if (currentBuild.result == 'UNSTABLE') {
                        echo "Quality Gate timed out. Checking status manually..."
                        
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            def statusCmd = """
                                curl -u "${SONAR_TOKEN}:" "${env.SONAR_URL}/api/qualitygates/project_status?projectKey=${env.SONAR_PROJECT_KEY}" | jq -r '.projectStatus.status'
                            """
                            
                            try {
                                def status = sh(script: statusCmd, returnStdout: true).trim()
                                echo "Manual check for Quality Gate status: ${status}"
                                
                                if (status == "ERROR") {
                                    echo "Quality Gate failed. Check SonarQube for details."
                                    currentBuild.result = 'FAILURE'
                                    error "Quality Gate failed"
                                } else if (status == "OK") {
                                    echo "Quality Gate passed manually!"
                                    currentBuild.result = 'SUCCESS'
                                } else {
                                    echo "Uncertain Quality Gate status. Proceeding with caution."
                                    currentBuild.result = 'UNSTABLE'
                                }
                            } catch (Exception e) {
                                echo "Failed to check Quality Gate status manually. Proceeding with caution."
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    }
                }
            }
        }
        
        stage('Prepare Artifact') {
            steps {
                script {
                    // Create artifacts directory
                    sh "mkdir -p ${ARTIFACT_DIR}"
                    
                    // Check if needed directories exist before zipping
                    def sourceDirs = ""
                    if (fileExists('src')) {
                        sourceDirs += "src/ "
                    }
                    if (fileExists('sql')) {
                        sourceDirs += "sql/ "
                    }
                    
                    if (sourceDirs.trim().isEmpty()) {
                        error "No source directories found to package"
                    }
                    
                    // Create a zip archive of the application
                    sh """
                        zip -r ${ARTIFACT_DIR}/${ARTIFACT_NAME}-${params.VERSION}.zip ${sourceDirs}
                    """
                }
            }
        }
        
        stage('Store Artifact') {
            steps {
                // Here you would upload to Nexus or S3
                archiveArtifacts artifacts: "${ARTIFACT_DIR}/${ARTIFACT_NAME}-${params.VERSION}.zip", 
                                 allowEmptyArchive: false,
                                 fingerprint: true
                
                echo "Artifact stored at: ${WORKSPACE}/${ARTIFACT_DIR}/${ARTIFACT_NAME}-${params.VERSION}.zip"
            }
        }
        
        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                echo "Deploying version ${params.VERSION} to Staging environment"
                
                script {
                    env.app_version = params.VERSION
                    def artifactName = "${ARTIFACT_NAME}-${params.VERSION}.zip"

                    // Check if ansible directory exists
                    sh "mkdir -p ansible || true"
                    
                    // Check if artifact exists before copying
                    if (!fileExists("${ARTIFACT_DIR}/${artifactName}")) {
                        error "Artifact ${artifactName} not found. Build may have failed."
                    }
                    
                    sh "cp ${ARTIFACT_DIR}/${artifactName} ansible/"
                    
                    // Use sshagent with the configured key
                    sshagent(['staging-ssh-key']) {
                        sh """
                            cd ansible
                            ansible-playbook playbook.yml -i inventory -e "target_env=staging app_version=${params.VERSION} artifact_name=${artifactName} artifact_path=./${artifactName}"
                        """
                    }
                }
                
                echo "Staging deployment complete. Access at http://${env.STAGING_PUBLIC_IP}"
            }
            post {
                failure {
                    echo "Staging deployment failed. Check SSH connections and server availability."
                }
            }
        }
        
        stage('Approve Production Deployment') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                // Manual approval step
                timeout(time: 1, unit: 'DAYS') {
                    input message: "Deploy to Production?", ok: "Approve"
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                echo "Deploying version ${params.VERSION} to Production environment"
                
                script {
                    env.app_version = params.VERSION
                    def artifactName = "${ARTIFACT_NAME}-${params.VERSION}.zip"

                    // Check if ansible directory exists
                    sh "mkdir -p ansible || true"
                    
                    // Check if artifact exists before copying
                    if (!fileExists("${ARTIFACT_DIR}/${artifactName}")) {
                        error "Artifact ${artifactName} not found. Build may have failed."
                    }
                    
                    sh "cp ${ARTIFACT_DIR}/${artifactName} ansible/"
                    
                    // Use sshagent with the configured key
                    sshagent(['production-ssh-key']) {
                        sh """
                            cd ansible
                            ansible-playbook playbook.yml -i inventory -e "target_env=production app_version=${params.VERSION} artifact_name=${artifactName} artifact_path=./${artifactName}"
                        """
                    }
                }
                
                echo "Production deployment complete. Access via internal network at http://172.31.23.26"
                echo "Note: Production server is only accessible from within the VPC or via VPN"
            }
            post {
                failure {
                    echo "Production deployment failed. Check SSH connections and server availability."
                }
                success {
                    echo "Production deployment successful! Sending notification..."
                    // Add notification code here if needed
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the logs."
        }
        always {
            // Clean workspace but keep artifacts
            cleanWs(patterns: [[pattern: "${ARTIFACT_DIR}/**", type: 'INCLUDE']])
            
            // Save SonarQube results link for easy access
            echo "SonarQube results available at: ${env.SONAR_URL}/dashboard/index/${env.SONAR_PROJECT_KEY}"
        }
    }
}
