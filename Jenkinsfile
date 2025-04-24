pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terraformtest"
        TF_STATE = "/home/shakil/terraformtest/terraform.tfstate"
        GIT_CREDENTIALS = credentials('github-key')
        MISTRAL_API_KEY = credentials('mistral-api-key')
        GIT_REPO = "https://github.com/shakilmunavary/AI-Powered-Jenkins-BuildFailure-Management.git"
        MISTRAL_API = "your-mistral-api-url"
    }
    stages {
        stage('Clone Repository') {
            steps {
                sh "rm -rf $TF_DIR && mkdir -p $TF_DIR"
                withCredentials([string(credentialsId: 'github-key', variable: 'GIT_TOKEN')]) {
                    sh "git clone https://$GIT_TOKEN@$GIT_REPO $TF_DIR"
                }
            }
        }
        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                dir(TF_DIR) {
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log | tee terraform_plan.log"
                    withCredentials([string(credentialsId: 'mistral-api-key', variable: 'API_KEY')]) {
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
                    sh "echo 'Are we okay to deploy'"
                }
            }
        }
    }
}
