pipeline {

    // Run this pipeline on the Jenkins agent/node named AGENT-1
    agent {
        label 'AGENT-1'
    }

    // Variables available in all stages
    environment {
        appVersion = ''              // Will be filled from package.json version
        REGION = "us-east-1"         // AWS region
        ACC_ID = "275785115817"      // AWS account ID
        PROJECT = "roboshop"         // Project name
        COMPONENT = "catalogue"      // Service/component name
    }

    // General pipeline rules
    options {
        timeout(time: 30, unit: 'MINUTES')   // Stop pipeline if it runs too long
        disableConcurrentBuilds()            // Do not allow another run at the same time
    }

    // Parameter shown when starting the job manually
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Run deploy after build')
    }

    // Build pipeline stages
    stages {

        stage('Read package.json') {
            steps {
                script {
                    // Read package.json file
                    def packageJson = readJSON file: 'package.json'

                    // Get the version from package.json and store it in appVersion
                    appVersion = packageJson.version

                    // Print the version in Jenkins console
                    echo "Package version: ${appVersion}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    // Install Node.js packages needed for the app
                    sh """
                        npm install
                    """
                }
            }
        }

        stage('Unit Testing') {
            steps {
                script {
                   sh """
                        echo "unit tests"
                    """
                }
            }
        }

        /* 
        // SonarQube code scan stage
        stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-7.2'   // Use Jenkins tool named sonar-7.2
            }
            steps {
                script {
                    // Connect to SonarQube server and run scan
                    withSonarQubeEnv(installationName: 'sonar-7.2') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        */

        /*
        // Enable webhook on SonarQube server and Wait for SonarQube quality gate result
        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true   // Fail pipeline if quality gate fails
                }
            }
        }
        */

        stage('Check Dependabot Alerts') {
            environment {
                GITHUB_TOKEN = credentials('github-token')   // Load GitHub token from Jenkins credentials
            }
            steps {
                script {
                    // Fetch alerts from GitHub
                    def response = sh(
                        script: """
                            curl -s -H "Accept: application/vnd.github+json" \
                                 -H "Authorization: token ${GITHUB_TOKEN}" \
                                 https://api.github.com/repos/daws-84s/catalogue/dependabot/alerts
                        """,
                        returnStdout: true
                    ).trim()

                    // Convert JSON text into Jenkins/Groovy object
                    def json = readJSON text: response

                    // Filter alerts by severity
                    def criticalOrHigh = json.findAll { alert ->
                        def severity = alert?.security_advisory?.severity?.toLowerCase()
                        def state = alert?.state?.toLowerCase()
                        return (state == "open" && (severity == "critical" || severity == "high"))
                    }

                    // Fail the pipeline if serious alerts exist
                    if (criticalOrHigh.size() > 0) {
                        error "❌ Found ${criticalOrHigh.size()} HIGH/CRITICAL Dependabot alerts. Failing pipeline!"
                    } else {
                        echo "✅ No HIGH/CRITICAL Dependabot alerts found."
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Use AWS credentials stored in Jenkins
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {

                        // Login to AWS ECR, build Docker image, then push it
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
                    }
                }
            }
        }

        stage('Check Scan Results') {
            steps {
                script {
                    // Use AWS credentials to check ECR image scan results
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {

                        // Get scan findings for the pushed Docker image
                        def findings = sh(
                            script: """
                                aws ecr describe-image-scan-findings \
                                --repository-name ${PROJECT}/${COMPONENT} \
                                --image-id imageTag=${appVersion} \
                                --region ${REGION} \
                                --output json
                            """,
                            returnStdout: true
                        ).trim()

                        // Convert JSON text into object
                        def json = readJSON text: findings

                        // Find only HIGH or CRITICAL vulnerabilities
                        def highCritical = json.imageScanFindings.findings.findAll {
                            it.severity == "HIGH" || it.severity == "CRITICAL"
                        }

                        // Fail build if serious vulnerabilities are found
                        if (highCritical.size() > 0) {
                            echo "❌ Found ${highCritical.size()} HIGH/CRITICAL vulnerabilities!"
                            currentBuild.result = 'FAILURE'
                            error("Build failed due to vulnerabilities")
                        } else {
                            echo "✅ No HIGH/CRITICAL vulnerabilities found."
                        }
                    }
                }
            }
        }

        stage('Trigger Deploy') {
            when {
                expression { params.deploy }   // Run this stage only if deploy=true
            }
            steps {
                script {
                    // Trigger another Jenkins job for deployment
                    build job: 'catalogue-cd',
                    parameters: [
                        string(name: 'appVersion', value: "${appVersion}"), // Send built version
                        string(name: 'deploy_to', value: 'dev')             // Deploy target environment
                    ],
                    propagate: false,  // If deploy job fails, this pipeline is not affected
                    wait: false        // Do not wait for deploy job to finish
                }
            }
        }
    }

    // Post section runs after all stages finish (can be based on build result)
    post {
        always {
            echo 'I will always say Hello again!'   // Runs every time
            deleteDir()                             // Clean workspace
        }
        success {
            echo 'Hello Success'                    // Runs only if pipeline succeeds
        }
        failure {
            echo 'Hello Failure'                    // Runs only if pipeline fails
        }
    }
}