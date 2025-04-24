pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terra-analyze-ai/terraform-ai-analytics"
        TF_STATE = "/home/shakil/terra-analyze-ai/terraform-ai-analytics/terraform.tfstate"
        GIT_CREDENTIALS = credentials('GitHubApiKey')
        MISTRAL_API_KEY = credentials('MISTRAL_API_KEY')
        GIT_REPO = "https://github.com/shakilmunavary/terraform-ai-analytics.git"
        MISTRAL_API = "https://api.mistral.ai/v1/chat/completions"
    }
    stages {
        stage('Clone Repository') {
            steps {
                sh "echo 'Started Working'"
                withCredentials([string(credentialsId: 'GitHubApiKey', variable: 'GIT_TOKEN')]) {
                    sh "git clone https://$GIT_TOKEN@$GIT_REPO $TF_DIR"
                }
            }
        }
        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                dir(TF_DIR) {
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log | tee terraform_plan.log"
                    withCredentials([string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY')]) {
                        sh """
                        curl -X POST $MISTRAL_API \\
                             -H 'Authorization: Bearer $API_KEY' \\
                             -H 'Content-Type: application/json' \\
                             -d '{
                                 "model": "mistral-large-latest",
                                 "prompt": "Analyze the Terraform plan and format the resource changes in a structured table with Resource Name, Action (Added/Deleted/Updated), and Estimated Cost."
                             }' > mistral_response.json
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
        stage('Manual Approval') {
            steps {
                input "Approve Terraform Deployment?"
            }
        }
        stage('Deploy Terraform Code') {
            steps {
                dir(TF_DIR) {
                    sh "terraform apply -auto-approve tfplan.log"
                }
            }
        }
    }
}
