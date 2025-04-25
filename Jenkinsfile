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
                        echo "TERRA FORM PLAN"
                                cd $TF_DIR/$GIT_REPO_NAME/terraform
                                terraform init
                                terraform plan -out=tfplan
                                terraform show -json tfplan > tfplan.json
                                PLAN_FILE_CONTENT=\$(cat tfplan.json)
                                ESCAPED_PLAN_FILE_CONTENT=\$(jq -Rs . <<< "\$PLAN_FILE_CONTENT")
                                
                                curl -X POST "https://api.mistral.ai/v1/chat/completions" \
                                     -H "Authorization: Bearer \$API_KEY" \
                                     -H "Content-Type: application/json" \
                                     -d '{
                                           "model": "mistral-large-latest",
                                           "messages": [
                                             { "role": "system", "content": "Analyze the terraform plan and recommend any suggestions. Also put all the resources in tabular format like Resource Name, Actions status Addition or Deletion or Update, Whats being changed, Cost) " },
                                             { "role": "user", "content": '"\$ESCAPED_PLAN_FILE_CONTENT"' }
                                           ],
                                           "max_tokens": 5000
                                         }' > ${TF_DIR}/ai_response.json
                                
                                # Read the JSON file
                                cd $TF_DIR
                                JSON_CONTENT=\$(cat ai_response.json)
                                
                                # Extract and format the 'content' field
                                FORMATTED_CONTENT=\$(echo "\$JSON_CONTENT" | jq '.choices[0].message.content')
                                
                                # Print the formatted content to the console
                                echo "\$FORMATTED_CONTENT"
                        """
                    }
                }
            }
        }
        stage('Parse & Display Mistral Recommendations') {
            steps {
                script {
                    def mistralResponse = readJSON(file: "${TF_DIR}/ai_response.json")
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
