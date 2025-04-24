pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terra-analyze-ai/"
        TF_STATE = "/home/shakil/terra-analyze-ai/terraform.tfstate"
        GIT_CREDENTIALS = credentials('GitHubApiKey')
        MISTRAL_API_KEY = credentials('MISTRAL_API_KEY')
        GIT_REPO = "https://github.com/shakilmunavary/terraform-ai-analytics.git"
        MISTRAL_API = "https://api.mistral.ai/v1/chat/completions"
        GIT_REPO_FOLDER="terraform-ai-analytics"
    }
    stages {
        stage('Checkout Terraform Code') {
            steps {
                sh "cd $TF_DIR/terraform"
                withCredentials([string(credentialsId: 'github-key', variable: 'GIT_TOKEN')]) {
                    sh "git clone https://$GIT_TOKEN@$GIT_REPO $TF_DIR"
                }
            }
        }

        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                dir(TF_DIR) {
                    sh "cd terraform-ai-analytics"
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log | tee terraform_plan.log"
                    withCredentials([string(credentialsId: 'MISTRAL_API_KEY', variable: 'MISTRAL_API_KEY')]) {
                        sh "curl -X POST -H 'Authorization: Bearer $MISTRAL_API_KEY' -d @terraform_plan.log $MISTRAL_API > mistral_response.json"
                    }
                }
            }
        }
        stage('Parse & Display Mistral Recommendations') {
            steps {
                script {
                    def mistralResponse = readJSON(file: "${TF_DIR}/mistral_response.json")
                    echo "========== Mistral AI Recommendations =========="
                    mistralResponse.recommendations.each { recommendation ->
                        echo "🔹 ${recommendation}"
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
                    sh "Deploying the Code"
                }
            }
        }
    }
}
