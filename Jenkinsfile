pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terra-analyze-ai/"
        GIT_REPO_NAME= "terraform-ai-analytics"
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
                dir("$TF_DIR/$GIT_REPO_NAME/terraform") {
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log "
                }

                script {
                    def tfPlanContent = readFile("${TF_DIR}/${GIT_REPO_NAME}/terraform/tfplan.log")
                    withCredentials([string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY')]) {
                        sh """
                        echo "TERRA FORM PLAN"
                        cat $tfPlanContent
                        curl -X POST $MISTRAL_API \\
                             -H 'Authorization: Bearer $API_KEY' \\
                             -H 'Content-Type: application/json' \\
                             -d '{
                                 "model": "mistral-large-latest",
                                 "prompt": "Analyze the Terraform plan and format the resource changes in a structured table with Resource Name, Action (Added/Deleted/Updated), and Estimated Cost.",
                                 "plan": "${tfPlanContent.replaceAll('\n', '\\n')}"
                             }' > ${TF_DIR}/mistral_response.json
                        """
                    }
                }
            }
        }
        stage('Parse & Display Mistral Recommendations') {
            steps {
                script {
                    def mistralResponse = readJSON(file: "${TF_DIR}/mistral_response.json")
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
