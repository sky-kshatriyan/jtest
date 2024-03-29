pipeline {
    agent any
    tools {
        "org.jenkinsci.plugins.terraform.TerraformInstallation" "terraform-v0.11.13"

    }
    parameters {
        string(name: 'SD_LAMBDA_URL', defaultValue: '', description: 'Lambda Function URL')
        string(name: 'WORKSPACE', defaultValue: 'development', description: 'workspace to use in Terraform')
    }
    environment {
        TF_HOME = tool('terraform-v0.11.13')
        TF_IN_AUTOMATION = 'true'
        DYNAMODB_STATELOCK = "sd-tfstatelock"
        NETWORKING_BUCKET = "sd-networking"
        NT_ACCESS_KEY = credentials('sd_nt_access_key')
        NT_SECRET_KEY = credentials('sd_nt_secret_key')
        
    }

    stages {
        stage('NetworkInit') {
            steps {
                dir('Scenario_12/03_network/'){
                    script {
                        ansiColor('xterm'){
                            if (fileExists(".terraform/terraform.tfstate")) {
                                sh "rm -rf .terraform/terraform.tfstate"
                            }
                            sh "terraform init -input=false -plugin-dir=/var/lib/jenkins/terraform_plugins \
                            --backend-config='dynamodb_table=$DYNAMODB_STATELOCK' --backend-config='bucket=$NETWORKING_BUCKET' \
                            --backend-config='access_key=$NT_ACCESS_KEY' --backend-config='secret_key=$NT_SECRET_KEY'"
                        }
                    }
                }
            }
        }
        stage('NetworkPlan') {
            steps {
                dir('Scenario_12/03_network/'){
                    script {
                        ansiColor('xterm'){
                            try {
                                sh "terraform workspace new ${params.WORKSPACE}"
                            } catch (err) {
                                sh "terraform workspace select ${params.WORKSPACE}"
                            }
                            sh "terraform plan -var 'aws_access_key=$NT_ACCESS_KEY' -var 'aws_secret_key=$NT_SECRET_KEY' \
                            -var 'url=${params.SD_LAMBDA_URL}' -out terraform-networking.tfplan;echo \$? > status"
                            stash name: "terraform-networking-plan", includes: "terraform-networking.tfplan"
                        }
                    }
                }
            }
        }
        stage('Networkapply') {
            steps {
                script{
                     ansiColor('xterm'){
                        def apply = false 
                        try{
                            input message: 'confirm apply', ok: 'Apply Config'
                            apply = true
                        } catch (err) {
                            apply = false
                            dir('Scenario_12/03_network/'){
                                sh "terraform destroy -var 'aws_access_key=$NT_ACCESS_KEY' -var 'aws_secret_key=$NT_SECRET_KEY' -var 'url=${params.SD_LAMBDA-_URL}' -force"
                            }
                            currentBuild.result = 'UNSTABLE'
                        }
                        if(apply){
                            dir('Scenario_12/03_network/'){
                                unstash "terraform-networking-plan"
                                sh 'terraform apply terraform-networking.tfplan'
                            }
                        }
                     }
                }    
            }
        }
    }
}

