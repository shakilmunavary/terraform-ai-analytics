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
                   cp  -rf $GIT_REPO_NAME $TF_DIR
                 """
            }
        }
        
        stage('Run Terraform Plan & Send to Mistral') {
            steps {
                    sh "cd $TF_DIR/$GIT_REPO_NAME"
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log | tee terraform_plan.log"
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
