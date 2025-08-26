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

        stage('Run Terraform Plan & Send to Azure OpenAI') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AZURE_API_KEY', variable: 'API_KEY'),
                        string(credentialsId: 'INFRACOST_APIKEY', variable: 'INFRACOST_API_KEY')
                    ]) {
                        sh """
                        #!/bin/bash
                        echo "Running Terraform Plan"
                        cd $TF_DIR/$GIT_REPO_NAME/terraform
                        terraform init
                        terraform plan -out=tfplan.binary
                        terraform show -json tfplan.binary > tfplan.json

                        echo "Running Infracost Breakdown"
                        infracost configure set api_key \$INFRACOST_API_KEY
                        infracost breakdown --path=tfplan.binary --format json --out-file totalcost.json

                        echo "Preparing AI Analysis"
                        touch ${TF_DIR}/ai_response.json
                        chmod 666 ${TF_DIR}/ai_response.json

                        PLAN_FILE_CONTENT=\$(cat tfplan.json | jq -Rs .)
                        GUARDRAILS_CONTENT=\$(cat $TF_DIR/$GIT_REPO_NAME/guardrails/guardrails.txt | jq -Rs .)

                        curl -s -X POST "$AZURE_API_BASE/openai/deployments/$DEPLOYMENT_NAME/chat/completions?api-version=$AZURE_API_VERSION" \\
                             -H "Content-Type: application/json" \\
                             -H "api-key: \$API_KEY" \\
                             -d '{
                                   "messages": [
                                     {
                                       "role": "system",
                                       "content": "You are a terraform expert. Analyze the terraform plan and compare it against the provided guardrails checklist. Return the following in plain HTML with no styles: 1) Whats Being Changed: tabular format with Resource Name, Type, Action (Add/Delete/Update), Details. 2) Terraform Code Recommendations. 3) Security and Compliance Recommendations. 4) Compliance Percentage: based on how many guardrails are met. 5) Overall Status: Pass if compliance >= 90%, else Fail."
                                     },
                                     {
                                       "role": "user",
                                       "content": "Terraform Plan:\\n" + \$PLAN_FILE_CONTENT + "\\n\\nGuardrails Checklist:\\n" + \$GUARDRAILS_CONTENT
                                     }
                                   ],
                                   "max_tokens": 10000,
                                   "temperature": 0.7
                                 }' > ${TF_DIR}/ai_response.json

                        jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                        """
                    }
                }
            }
        }
    }

    post {
        always {
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
}
