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
                                             { "role": "system", "content": "Convert the following JSON to HTML format." },
                                             { "role": "user", "content": '"\$PLAN_FILE_CONTENT"' }
                                           ],
                                           "max_tokens": 5000
                                         }' > ${TF_DIR}/ai_response.html
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            publishHTML([
                reportName: 'Mistral AI Response',
                reportDir: "${TF_DIR}",
                reportFiles: 'ai_response.html',
                keepAll: true,
                allowMissing: false,
                alwaysLinkToLastBuild: true
            ])
        }
    }
}
