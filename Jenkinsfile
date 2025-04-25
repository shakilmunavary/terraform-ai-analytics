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
                                
                                # Ensure jq is installed
                                if ! command -v jq &> /dev/null
                                then
                                    echo "jq could not be found, installing..."
                                    sudo apt-get update
                                    sudo apt-get install -y jq
                                fi
                                
                                # Read the JSON file content
                                PLAN_FILE_CONTENT=\$(cat tfplan.json | jq -Rs .)
                                
                                curl -X POST "https://api.mistral.ai/v1/chat/completions" \
                                     -H "Authorization: Bearer \$API_KEY" \
                                     -H "Content-Type: application/json" \
                                     -d '{
                                           "model": "mistral-large-latest",
                                           "messages": [
                                             { "role": "system", "content": "Analyze the terraform plan and recommend any suggestions. Also put all the resources in tabular format like Resource Name, Actions status Addition or Deletion or Update, Whats being changed, Cost) " },
                                             { "role": "user", "content": '"\$PLAN_FILE_CONTENT"' }
                                           ],
                                           "max_tokens": 5000
                                         }' > ${TF_DIR}/ai_response.json
                                cd $TF_DIR/
                                AI_REPONSE_JSON=\$(cat ai_response.json | jq -Rs .)
                                
                                curl -X POST "https://api.mistral.ai/v1/chat/completions" \
                                     -H "Authorization: Bearer \$API_KEY" \
                                     -H "Content-Type: application/json" \
                                     -d '{
                                           "model": "mistral-large-latest",
                                           "messages": [
                                             { "role": "system", "content": "Please read the json file and give me an html equvalent file for better readability " },
                                             { "role": "user", "content": '"\$AI_REPONSE_JSON"' }
                                           ],
                                           "max_tokens": 5000
                                         }' > ${TF_DIR}/ai_response.html



                        """
                    }
                }
            }
        }
        stage('Parse & Display Mistral Recommendations') {
            steps {
                script {
                    def mistralResponse = readJSON(file: "${TF_DIR}/ai_response.html")
                    echo "========== Mistral AI Recommendations =========="
                    echo String.format("%-30s %-15s %-10s", "Resource", "Action", "Estimated Cost")
                    mistralResponse.recommendations.each { recommendation ->
                        echo String.format("%-30s %-15s %-10s", recommendation.resource, recommendation.action, recommendation.cost)
                    }
                }
            }
        }
    }
}
