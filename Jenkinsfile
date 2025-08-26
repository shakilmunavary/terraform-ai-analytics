pipeline {
    agent any
    environment {
        TF_DIR = "/home/AI-SDP-PLATFORM/terra-analysis/"
        GIT_REPO_NAME = "terraform-ai-analytics"
        TF_STATE = "/home/AI-SDP-PLATFORM/terra-analysis/terraform-ai-analytics/terraform.tfstate"
        MISTRAL_API_KEY = credentials('MISTRAL_API_KEY')
        MISTRAL_API = "https://api.mistral.ai/v1/chat/completions"
        INFRACOST_APIKEY = credentials('INFRACOST_APIKEY')
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
                    cp -rf $GIT_REPO_NAME $TF_DIR
                """
            }
        }

        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY'),
                        string(credentialsId: 'INFRACOST_APIKEY', variable: 'INFRACOST_API_KEY')
                    ]) {
                        sh """
                        #!/bin/bash
                        echo "Running Terraform Plan"
                        cd $TF_DIR/$GIT_REPO_NAME/terraform
                        terraform init
                        terraform plan -out=tfplan.binary
                        terraform show -json tfplan.binary > tfplan.json

                        SAMPLE_HTML=\$(cat /home/AI-SDP-PLATFORM/terra-analysis/sample.html | jq -Rs .)
                        PLAN_JSON=\$(cat tfplan.json | jq -Rs .)

                        sleep 5

                        curl -X POST "$MISTRAL_API" \\
                             -H "Authorization: Bearer \$API_KEY" \\
                             -H "Content-Type: application/json" \\
                             -d '{
                                   "model": "mistral-large-latest",
                                   "messages": [
                                     {
                                       "role": "system",
                                       "content": "You are an expert Terraform analyst. Analyze the provided terraform plan and generate a response in HTML format that mimics the structure, theme, and styling of the reference HTML provided. The output should contain two sections: (1) What’s Being Changed — a table listing Resource Name, Action (Add/Update/Delete), and Details of Change; (2) Recommendations — a styled section with improvement suggestions. Do not return JSON or markdown. Only return valid, styled HTML content similar to the reference. (3) Over all Complaince Percentage (4) Overall Status of pipline, display Reject if Compliance is below 80%"
                                     },
                                     {
                                       "role": "user",
                                       "content": "Reference HTML for formatting: \$SAMPLE_HTML"
                                     },
                                     {
                                       "role": "user",
                                       "content": "Terraform Plan JSON: \$PLAN_JSON"
                                     }
                                   ],
                                   "max_tokens": 5000
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
