pipeline {
    parameters {
        choice(name: 'ACTION', choices: ['APPLY', 'DESTROY'], description: 'Choose action to perform')
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run the selected action after generating plan?')
    }
    /*environment {
        //VAULT_ADDR = credentials('vaultUrl')
        //VAULT_TOKEN = credentials('vaultCred')
        //AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        //AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    }*/

    agent {
        docker { 
            image 'vidaldocker/mastertf:kn1.0.2' 
        }
    }

    stages {
        stage('Clean Workspace') {
            steps {
                script {
                    echo 'Cleaning workspace...'
                    sh 'find . -maxdepth 1 ! -name . -exec rm -rf {} +'
                }
            }
        }
        stage('Terraform and Checkout') {
            parallel {
                stage('Verify Docker Setup') {
                    steps {
                        sh 'terraform --version'  // Ensure Terraform command works
                    }
                }

                stage('Checkout') {
                    steps {
                        script {
                            dir("dynamic-tf") {
                                git "https://github.com/vidalgithub/terraform-aws-vault-dynamic.git"
                            }
                        }
                    }
                }
            }
        }

        stage('Plan') {
            when {
                expression { return params.ACTION == 'APPLY' }
            }
            steps {
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultUrl: 'env.vaultUrl'], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[vaultKey: 'VAULT_ADDR'], [vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'
                        sh 'terraform init'
                        sh 'terraform plan -out=tfplan'
                        sh 'terraform show -no-color tfplan > tfplan.txt'
                    }
                }
              }
            }
        }

        stage('Plan Destroy') {
            when {
                expression { return params.ACTION == 'DESTROY' }
            }
            steps {
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultUrl: 'env.vaultUrl'], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[vaultKey: 'VAULT_ADDR'], [vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'                        
                        sh 'terraform init'
                        sh 'terraform plan -destroy -out=tfplan-destroy'
                        sh 'terraform show -no-color tfplan-destroy > tfplan-destroy.txt'
                    }
                }
              }
            }
        }

        stage('Approval') {
            when {
                not {
                    equals expected: true, actual: params.autoApprove
                }
            }
            steps {
                script {
                    def planFile = params.ACTION == 'APPLY' ? 'terraform/tfplan.txt' : 'terraform/tfplan-destroy.txt'
                    def plan = readFile planFile
                    input message: "Do you want to ${params.ACTION.toLowerCase()} the resources?",
                          parameters: [text(name: 'Plan', description: 'Please review the plan', defaultValue: plan)]
                }
            }
        }

        stage('Apply') {
            when {
                expression { return params.ACTION == 'APPLY' }
            }
            steps {
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultUrl: 'env.vaultUrl'], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[vaultKey: 'VAULT_ADDR'], [vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'                        
                        sh 'terraform apply -input=false tfplan'
                    }
                }
              }
            }
        }

        stage('Destroy') {
            when {
                expression { return params.ACTION == 'DESTROY' }
            }
            steps {
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultUrl: 'env.vaultUrl'], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[vaultKey: 'VAULT_ADDR'], [vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'                        
                        sh 'terraform apply -destroy -input=false tfplan-destroy'
                    }
                }
              }
            }
        }
    }
}
