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
                   pwd
                   ls -ltr
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
                        echo "TERRA FORM PLAN"
                        cd $TF_DIR/$GIT_REPO_NAME/terraform
                        terraform init
                        terraform plan -out=tfplan.binary
                        terraform show -json tfplan.binary > tfplan.json

                        touch ${TF_DIR}/ai_response.json
                        chmod 666 ${TF_DIR}/ai_response.json
                        
                        PLAN_FILE_CONTENT=\$(cat tfplan.json | jq -Rs .)
                        # Add sleep to reduce burst pressure
                        sleep 5

                        curl -X POST "$MISTRAL_API" \
                             -H "Authorization: Bearer \$API_KEY" \
                             -H "Content-Type: application/json" \
                             -d '{
                                   "model": "mistral-large-latest",
                                   "messages": [
                                     { "role": "system", "content": "Analyze the terraform plan and recommend any suggestions. Also put all the resources in tabular format like Resource Name, Actions status Addition or Deletion or Update, Whats being changed.Review this inputs give me your response content in beautiful html format with good theme and look and feel instead of json. First Sections Whats Being Changed second Sections Section Recomendations. Just give me only html content as response dont give me any additional contents." },
                                     { "role": "user", "content": '"\$PLAN_FILE_CONTENT"' }
                                   ],
                                   "max_tokens": 5000
                                 }' > ${TF_DIR}/ai_response.json
                        
                        sleep 5
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
