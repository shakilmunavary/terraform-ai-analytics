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
                    sh "terraform plan > tfplan.log "
                }

                script {
                    withCredentials([string(credentialsId: 'MISTRAL_API_KEY', variable: 'API_KEY')]) {
                        sh """
                        echo "TERRA FORM PLAN"
                        curl -X POST $MISTRAL_API \\
                             -H 'Authorization: Bearer $API_KEY' \\
                             -H 'Content-Type: multipart/form-data' \\
                             -F "model=mistral-large-latest" \\
                             -F "prompt=Analyze the Terraform plan and give recommendation. And also format the resource changes in a structured table with Resource Name, Action (Added/Deleted/Updated), and Estimated Cost." \\
                             -F "file=@$TF_DIR/$GIT_REPO_NAME/terraform/tfplan.log" \\
                             > ${TF_DIR}/mistral_response.json
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
