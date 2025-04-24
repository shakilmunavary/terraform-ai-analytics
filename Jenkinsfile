pipeline {
    agent any
    environment {
        TF_DIR = "/home/shakil/terraformtest"
        TF_STATE = "/home/shakil/terraformtest/terraform.tfstate"
        GIT_CREDENTIALS = credentials('github-key')
        MISTRAL_API_KEY = credentials('mistral-api-key')
        GIT_REPO = "your-github-repo-url"
        MISTRAL_API = "your-mistral-api-url"
    }
    stages {
        stage('Checkout Terraform Code') {
            steps {
                sh "rm -rf $TF_DIR && mkdir -p $TF_DIR"
                withCredentials([string(credentialsId: 'github-key', variable: 'GIT_TOKEN')]) {
                    sh "git clone https://$GIT_TOKEN@$GIT_REPO $TF_DIR"
                }
            }
        }
        stage('Run Terraform Plan') {
            steps {
                dir(TF_DIR) {
                    sh "terraform init"
                    sh "terraform plan -out=tfplan.log | tee terraform_plan.log"
                    withCredentials([string(credentialsId: 'mistral-api-key', variable: 'API_KEY')]) {
                        sh "curl -X POST -H 'Authorization: Bearer $API_KEY' -d @terraform_plan.log $MISTRAL_API"
                    }
                }
            }
        }
        stage('Parse & Display Plan Output') {
            steps {
                script {
                    def output = sh(script: "terraform show -json tfplan.log", returnStdout: true).trim()
                    def parsedData = parseTerraformOutput(output)
                    printFormattedOutput(parsedData)
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

def parseTerraformOutput(jsonOutput) {
    def parsed = readJSON(text: jsonOutput)
    def resources = []
    parsed.resource_changes.each { resource ->
        resources << [resource.name, resource.change.actions.join(", "), resource.change.before_cost, resource.change.after_cost]
    }
    return resources
}

def printFormattedOutput(parsedData) {
    echo "========== Terraform Plan Summary =========="
    echo String.format("%-30s %-15s %-10s %-10s", "Resource", "Action", "Old Cost", "New Cost")
    parsedData.each {
        echo String.format("%-30s %-15s %-10s %-10s", it[0], it[1], it[2], it[3])
    }
}
