pipeline {
    agent any
    environment {
        TF_DIR = "/home/AI-SDP-PLATFORM/terra-analysis/"
        GIT_REPO_NAME = "terraform-ai-analytics"
        TF_STATE = "/home/AI-SDP-PLATFORM/terra-analysis/terraform-ai-analytics/terraform.tfstate"
        MISTRAL_API_KEY = credentials('MISTRAL_API_KEY')
        MISTRAL_API = "https://api.mistral.ai/v1/chat/completions"
    }

    stages {
        stage('Clone Repository') {
            steps {
                sh '''
                    rm -rf ${GIT_REPO_NAME}
                    git clone https://github.com/shakilmunavary/terraform-ai-analytics.git
                    sleep 3
                '''
            }
        }

        stage('Move To Working Directory') {
            steps {
                sh '''
                    cp -rf ${GIT_REPO_NAME} ${TF_DIR}
                    sleep 3
                '''
            }
        }

        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY')
                    ]) {
                        sh '''
                        echo "Running Terraform Plan"
                        cd ${TF_DIR}/${GIT_REPO_NAME}/terraform
                        terraform init
                        sleep 5
                        terraform plan -out=tfplan.binary
                        sleep 5
                        terraform show -json tfplan.binary > tfplan.json
                        sleep 5

                        which jq || { echo "jq not found"; exit 1; }
                        PLAN_JSON=$(jq -Rs . tfplan.json)

                        cat <<EOF > payload.json
{
  "model": "mistral-large-2411",
  "messages": [
    {
      "role": "system",
      "content": "You are an expert Terraform analyst. Analyze the provided terraform plan and generate a response output which should contain five sections: (1) What’s Being Changed — a table listing Resource Name, Action (Add/Update/Delete), and Details of Change; (2) Terraform Code Recommendations; (3) Security and Compliance Section — Provide all security and compliance recommendations; (4) Compliance Percentage — Include an overall compliance percentage based on best practices; (5) Overall Pipeline Status — if compliance is above 90%, mark as Pass, else Fail. Do not return JSON or markdown. Only return valid, simple plain HTML as output and don't add any styles."
    },
    {
      "role": "user",
      "content": "Terraform Plan JSON: ${PLAN_JSON}"
    }
  ],
  "max_tokens": 20000
}
EOF

                        curl -X POST "${MISTRAL_API}" \
                             -H "Authorization: Bearer ${API_KEY}" \
                             -H "Content-Type: application/json" \
                             -d @payload.json > ${TF_DIR}/ai_response.json
                        sleep 10

                        if [ ! -s ${TF_DIR}/ai_response.json ]; then
                          echo "API failed after retries. Writing fallback HTML..."
                          echo "<html><body><h2>AI Analysis Failed: Service Tier Capacity Exceeded</h2><p>Please try again later or upgrade your API tier.</p></body></html>" > ${TF_DIR}/output.html
                        else
                          jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                        fi

                        sleep 5
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            sleep(time: 5, unit: 'SECONDS')
            publishHTML([
                reportName: 'AI Analysis',
                reportDir: "${env.TF_DIR}",
                reportFiles: 'output.html',
                keepAll: true,
                allowMissing: false,
                alwaysLinkToLastBuild: true
            ])
        }
    }
}
