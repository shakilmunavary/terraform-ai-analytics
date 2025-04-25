pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terra-analyze-ai/"
        GIT_REPO_NAME = "terraform-ai-analytics"
        TF_STATE = "/home/shakil/terra-analyze-ai/terraform-ai-analytics/terraform.tfstate"
        MISTRAL_API_KEY = credentials('MISTRAL_API_KEY')
        MISTRAL_API = "https://api.mistral.ai/v1/chat/completions"
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
                    withCredentials([string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY')]) {
                        sh """
                        #!/bin/bash
                        echo "TERRA FORM PLAN"
                                cd $TF_DIR/$GIT_REPO_NAME/terraform
                                terraform init
                                terraform plan -out=tfplan
                                terraform show -json tfplan > tfplan.json
                                
                                # Ensure jq is installed and has execute permissions
                                if ! command -v jq &> /dev/null
                                then
                                    echo "jq could not be found, installing..."
                                    sudo apt-get update
                                    sudo apt-get install -y jq
                                fi
                                sudo chmod +x /usr/bin/jq
                                
                                # Ensure the ai_response.json file has the correct permissions
                                touch ${TF_DIR}/ai_response.json
                                chmod 666 ${TF_DIR}/ai_response.json
                                
                                # Read the JSON file content
                                PLAN_FILE_CONTENT=\$(cat tfplan.json | jq -Rs .)
                                
                                curl -X POST "https://api.mistral.ai/v1/chat/completions" \
                                     -H "Authorization: Bearer \$API_KEY" \
                                     -H "Content-Type: application/json" \
                                     -d '{
                                           "model": "mistral-large-latest",
                                           "messages": [
                                             { "role": "system", "content": "Analyze the terraform plan and recommend any suggestions. Also put all the resources in tabular format like Resource Name, Actions status Addition or Deletion or Update, Whats being changed, Cost. Give me your response content in html format instead of json so that I can parse it easy.) " },
                                             { "role": "user", "content": '"\$PLAN_FILE_CONTENT"' }
                                           ],
                                           "max_tokens": 5000
                                         }' > ${TF_DIR}/ai_response.json
                                
                                # Extract HTML content and save as output.html
                                jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Print the HTML content to the console
                sh "cat ${TF_DIR}/output.html"
            }
        }
    }
}
