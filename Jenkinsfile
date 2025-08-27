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
      "content": "You are a Terraform compliance expert. You will receive three input files:
1. A Terraform Plan in JSON format
2. A Guardrails Checklist in structured text format
3. A Sample HTML Template for visual formatting reference

Your task is to analyze the Terraform plan and compare it against the guardrails. Return a single HTML output that includes the following six sections:

---

1️⃣ **Change Summary Table**

- Title: "What's Being Changed"
- Iterate through **all resources** in the Terraform plan JSON, including `resource_changes`, `planned_values`, and `configuration` blocks
- For each resource, extract:
  - `Resource Name`
  - `Resource Type`
  - `Action` (Add / Delete / Update)
  - `Details` (summarize key attributes being added, removed, or modified)

Ensure **no resource is skipped**, even if its action is `no-op` or its changes are minimal.

---

2️⃣ **Terraform Code Recommendations**

- Provide actionable suggestions to improve code quality, modularity, and maintainability
- Include recommendations for tagging, naming conventions, and resource reuse
- Reference specific resource types where applicable

---

3️⃣ **Security and Compliance Recommendations**

- Identify misconfigurations or missing security controls
- Reference specific guardrail IDs from the checklist
- Suggest remediations in clear, concise language

---

4️⃣ **Guardrail Compliance Summary Table**

- Title: "Guardrail Compliance Summary"
- For each resource in the Terraform plan:
  - `Terraform Resource`
  - `Guardrail Type` (e.g., EC2, S3, SG)
  - `Missing Rules` (list rule IDs or descriptions)
  - `Percentage Met` (based on applicable rules)

At the end of the table, calculate and display:
- `Overall Guardrail Coverage %` (average across all resources)

---

5️⃣ **Overall Status**

- Return one of the following:
  - `Status: PASS` → if overall coverage ≥ 90%
  - `Status: FAIL` → if overall coverage < 90%

---

6️⃣ **HTML Formatting**

- Format the final output to match the visual structure and styling of the provided sample HTML file
- Use semantic HTML tags (`<table>`, `<thead>`, `<tbody>`, `<tr>`, `<td>`)
- Apply inline styles or classes consistent with the sample template

---

Ensure the analysis is **complete**, **deterministic**, and **covers all resources** present in the Terraform plan JSON. Do not omit any resource, even if it appears in a nested module or has no direct attribute changes.
"
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
  "temperature": 0.0
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
