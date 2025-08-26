
pipeline {
    agent any

    environment {
        TF_DIR           = "/home/AI-SDP-PLATFORM/terra-analysis/"
        GIT_REPO_NAME    = "terraform-ai-analytics"
        TF_STATE         = "/home/AI-SDP-PLATFORM/terra-analysis/terraform-ai-analytics/terraform.tfstate"
        DEPLOYMENT_NAME  = "gpt-4o"
        AZURE_API_BASE   = credentials('AZURE_API_BASE')
        AZURE_API_VERSION = "2025-01-01-preview"
        AZURE_API_KEY = credentials('AZURE_API_KEY')
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

        stage('Run Terraform Plan & Send to Azure OpenAI') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AZURE_API_KEY', variable: 'API_KEY'),
                        string(credentialsId: 'INFRACOST_APIKEY', variable: 'INFRACOST_API_KEY')
                    ]) {
                        sh """
                        #!/bin/bash
                        echo "Running Terraform Plan"
                        cd $TF_DIR/$GIT_REPO_NAME/terraform
                        terraform init
                        terraform plan -out=tfplan.binary
                        terraform show -json tfplan.binary > tfplan.json

                        echo "Running Infracost Breakdown"
                        infracost configure set api_key \$INFRACOST_API_KEY
                        infracost breakdown --path=tfplan.binary --format json --out-file totalcost.json

                        echo "Preparing AI Analysis"
                        touch ${TF_DIR}/ai_response.json
                        chmod 666 ${TF_DIR}/ai_response.json

                        PLAN_FILE_CONTENT=\$(cat tfplan.json | jq -Rs .)

                        curl -s -X POST "$AZURE_API_BASE/openai/deployments/$DEPLOYMENT_NAME/chat/completions?api-version=$AZURE_API_VERSION" \\
                             -H "Content-Type: application/json" \\
                             -H "api-key: \$API_KEY" \\
                             -d '{
                                   "messages": [
                                     { "role": "system", "content": "You are a terraform expert. Analyze the terraform plan attached and give the below response in plain html with no styles.1)Whats Being Changed : here put the data in the tablur format like Resource Name, Type , Action (Add, Delete , Update), More Details 2)Terraform Code Recommendations : here provide any recomendations if needed in the terraform code.3)Security and Compliance : Provide recomendations on Security and complaince in the terraform plan.5)Compliance Percentage : Overall Percentage of compliance met 6)Overall Status : Pass / Fail. Compliance Above 90% should be pass else fail." },
                                     { "role": "user", "content": '"\$PLAN_FILE_CONTENT"' }
                                   ],
                                   "max_tokens": 10000,
                                   "temperature": 0.7
                                 }' > ${TF_DIR}/ai_response.json

                        jq -r '.choices[0].message.content' ${TF_DIR}/ai_response.json > ${TF_DIR}/output.html
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            publishHTML([
                reportName: 'AI Analysis',
                reportDir: "${TF_DIR}",
                reportFiles: 'output.html',
                keepAll: true,
                allowMissing: false,
                alwaysLinkToLastBuild: true
            ])
        }
    }
}
