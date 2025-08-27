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
                    PLAN_FILE_CONTENT=\$(jq -Rs . < tfplan.json)
                    GUARDRAILS_CONTENT=\$(jq -Rs . < $TF_DIR/$GIT_REPO_NAME/guardrails/guardrails.txt)
                    SAMPLE_HTML=\$(jq -Rs . < $TF_DIR/$GIT_REPO_NAME/sample.html)

                    cat <<EOF > ${TF_DIR}/payload.json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a terraform expert. You will receive three files: one is a Terraform plan in JSON format, and the other is a guardrails checklist and another one is sample html file. Your job is to analyze the plan and compare it against the guardrails. Return the following in HTML : 1) Whats Being Changed: tabular format with Resource Name, Type, Action (Add/Delete/Update), Details. 2) Terraform Code Recommendations. 3) Security and Compliance Recommendations. 4) Compliance Percentage: For every resource in terraform plan check if guard rail exits and give me a tabuler data like Terraform Resource | Guardrail Type (Ec2,S3)| Missing Rules | Percentage Met. End of the table put overall percentage coverage of gurardails. 5) Overall Status: Pass if over al compliance >= 90%, else Fail. Finally convert the simple html file to follow same format as the attached sample html file"
    },
    {
      "role": "user",
      "content": "Terraform Plan File:\\n"
    },
    {
      "role": "user",
      "content": \${PLAN_FILE_CONTENT}
    },

    {
      "role": "user",
      "content": "Sample HTML File:\\n"
    },
    {
      "role": "user",
      "content": \${SAMPLE_HTML}
    },

    {
      "role": "user",
      "content": "Guardrails Checklist File:\\n"
    },
    {
      "role": "user",
      "content": \${GUARDRAILS_CONTENT}
    }
  ],
  "max_tokens": 10000,
  "temperature": 0.7
}
EOF

                    echo "Calling Azure OpenAI"
                    curl -s -X POST "$AZURE_API_BASE/openai/deployments/$DEPLOYMENT_NAME/chat/completions?api-version=$AZURE_API_VERSION" \\
                         -H "Content-Type: application/json" \\
                         -H "api-key: \$API_KEY" \\
                         -d @${TF_DIR}/payload.json > ${TF_DIR}/ai_response.json

                    jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                    """
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
