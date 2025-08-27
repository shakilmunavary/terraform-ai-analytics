pipeline {
    agent any

    environment {
        TF_DIR            = "/home/AI-SDP-PLATFORM/terra-analysis/"
        GIT_REPO_NAME     = "terraform-ai-analytics"
        TF_STATE          = "/home/AI-SDP-PLATFORM/terra-analysis/terraform-ai-analytics/terraform.tfstate"
        DEPLOYMENT_NAME   = "gpt-4o"
        AZURE_API_BASE    = credentials('AZURE_API_BASE')
        AZURE_API_VERSION = "2025-01-01-preview"
        AZURE_API_KEY     = credentials('AZURE_API_KEY')
    }

    stages {
        stage('Clone Repository') {
            steps {
                sh """
                    rm -rf $GIT_REPO_NAME
                    git clone 'https://github.com/shakilmunavary/terraform-ai-analytics.git'
                """
            }
        }

        stage('Move To Working Directory') {
            steps {
                sh """
                    pwd
                    ls -ltr
                    cp -rf $GIT_REPO_NAME $TF_DIR
                """
            }
        }

        stage('Run Terraform Plan & Infracost') {
            steps {
                withCredentials([
                    string(credentialsId: 'AZURE_API_KEY', variable: 'API_KEY'),
                    string(credentialsId: 'INFRACOST_APIKEY', variable: 'INFRACOST_API_KEY')
                ]) {
                    sh """
                        echo "Running Terraform Plan"
                        cd $TF_DIR/$GIT_REPO_NAME/terraform
                        terraform init
                        terraform plan -out=tfplan.binary
                        terraform show -json tfplan.binary > tfplan.json

                        echo "Running Infracost Breakdown"
                        infracost configure set api_key $INFRACOST_API_KEY
                        infracost breakdown --path=tfplan.binary --format json --out-file totalcost.json
                    """
                }
            }
        }

        stage('Prepare AI Payload') {
            steps {
                script {
                    def planContent = sh(script: "jq -Rs . < ${TF_DIR}/${GIT_REPO_NAME}/terraform/tfplan.json", returnStdout: true).trim()
                    def guardrailsContent = sh(script: "jq -Rs . < ${TF_DIR}/${GIT_REPO_NAME}/guardrails/guardrails.txt", returnStdout: true).trim()
                    def sampleHtml = sh(script: "jq -Rs . < ${TF_DIR}/${GIT_REPO_NAME}/sample.html", returnStdout: true).trim()

                    def payload = [
                        messages: [
                            [
                                role: "system",
                                content: """You are a Terraform compliance expert. You will receive three input files:
1. A Terraform Plan in JSON format
2. A Guardrails Checklist in structured text format
3. A Sample HTML Template for visual formatting reference

Your task is to analyze the Terraform plan and compare it against the guardrails. Return a single HTML output that includes the following six sections:

1️⃣ Change Summary Table
2️⃣ Terraform Code Recommendations
3️⃣ Security and Compliance Recommendations
4️⃣ Guardrail Compliance Summary Table
5️⃣ Overall Status
6️⃣ HTML Formatting"""
                            ],
                            [ role: "user", content: "Terraform Plan File:\\n" ],
                            [ role: "user", content: planContent ],
                            [ role: "user", content: "Sample HTML File:\\n" ],
                            [ role: "user", content: sampleHtml ],
                            [ role: "user", content: "Guardrails Checklist File:\\n" ],
                            [ role: "user", content: guardrailsContent ]
                        ],
                        max_tokens: 10000,
                        temperature: 0.0
                    ]

                    writeFile file: "${TF_DIR}/payload.json", text: groovy.json.JsonOutput.toJson(payload)
                }
            }
        }

        stage('Call Azure OpenAI') {
            steps {
                sh """
                    echo "Calling Azure OpenAI"
                    curl -s -X POST "$AZURE_API_BASE/openai/deployments/$DEPLOYMENT_NAME/chat/completions?api-version=$AZURE_API_VERSION" \\
                         -H "Content-Type: application/json" \\
                         -H "api-key: $AZURE_API_KEY" \\
                         -d @${TF_DIR}/payload.json > ${TF_DIR}/ai_response.json

                    jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                """
            }
        }

        stage('Publish AI Analysis Report') {
            steps {
                publishHTML([
                    reportName: 'AI Analysis',
                    reportDir: "${TF_DIR}",
                    reportFiles: 'output.html',
                    keepAll: true,
                    allowMissing: false,
                    alwaysLinkToLastBuild: true
                ])
            }
        }

        stage('Manual Validation') {
            steps {
                script {
                    def userInput = input(
                        id: 'userApproval', message: 'Compliance Validation Result',
                        parameters: [
                            choice(name: 'Decision', choices: ['Approve', 'Reject'], description: 'Select action based on compliance report')
                        ]
                    )

                    if (userInput == 'Approve') {
                        currentBuild.description = "Approved by user"
                        env.PIPELINE_DECISION = 'APPROVED'
                    } else {
                        currentBuild.description = "Rejected by user"
                        env.PIPELINE_DECISION = 'REJECTED'
                    }
                }
            }
        }

        stage('Approve Stage') {
            when {
                expression { env.PIPELINE_DECISION == 'APPROVED' }
            }
            steps {
                echo "✅ Pipeline approved. Proceeding with deployment or next steps..."
                // Add deployment logic here
            }
        }

        stage('Reject Stage') {
            when {
                expression { env.PIPELINE_DECISION == 'REJECTED' }
            }
            steps {
                echo "❌ Pipeline rejected. Halting further actions."
                // Add rejection handling here
            }
        }
    }
}
