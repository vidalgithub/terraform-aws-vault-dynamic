pipeline {
    parameters {
        choice(name: 'ACTION', choices: ['APPLY', 'DESTROY'], description: 'Choose action to perform')
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run the selected action after generating plan?')
    }

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
                            // Clone the repository manually with a single branch option
                            sh 'git clone --branch main https://github.com/vidalgithub/terraform-aws-vault-dynamic.git dynamic-tf'
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
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultCredentialId: 'vaultCred', vaultUrl: env.vaultUrl], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[envVar: 'VAULT_ADDR', vaultKey: 'VAULT_ADDR'], [envVar: 'VAULT_TOKEN', vaultKey: 'VAULT_TOKEN']]]]) {
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
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultCredentialId: 'vaultCred', vaultUrl: env.vaultUrl], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[envVar: 'VAULT_ADDR', vaultKey: 'VAULT_ADDR'], [envVar: 'VAULT_TOKEN', vaultKey: 'VAULT_TOKEN']]]]) {
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
                    def planFile = params.ACTION == 'APPLY' ? 'dynamic-tf/tfplan.txt' : 'dynamic-tf/tfplan-destroy.txt'
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
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultCredentialId: 'vaultCred', vaultUrl: env.vaultUrl], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[envVar: 'VAULT_ADDR', vaultKey: 'VAULT_ADDR'], [envVar: 'VAULT_TOKEN', vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'
                        try {
                            sh 'terraform apply -input=false tfplan'
                        } catch (Exception e) {
                            def errorMessage = e.getMessage()
                            echo 'Ignoring errors during apply. Check the plan for issues.'
                            echo errorMessage
                            // Check for specific error messages and decide whether to skip apply
                            if (errorMessage.contains('User with name aws-admin-vaut already exists') ||
                                errorMessage.contains('BucketAlreadyExists') ||
                                errorMessage.contains('Table already exists') ||
                                errorMessage.contains('resource record set already exists')) {
                                echo 'Resource already exists. Skipping apply.'
                                currentBuild.result = 'SUCCESS'
                            } else {
                                error 'Unknown error occurred during apply: ' + errorMessage
                            }
                        }
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
              withVault(configuration: [disableChildPoliciesOverride: false, timeout: 60, vaultCredentialId: 'vaultCred', vaultUrl: env.vaultUrl], vaultSecrets: [[path: 'mycreds/vault-server1/vault-creds', secretValues: [[envVar: 'VAULT_ADDR', vaultKey: 'VAULT_ADDR'], [envVar: 'VAULT_TOKEN', vaultKey: 'VAULT_TOKEN']]]]) {
                script {
                    dir("dynamic-tf") {
                        sh ' export VAULT_ADDR="${VAULT_ADDR}"'
                        sh ' export VAULT_TOKEN="${VAULT_TOKEN}"'                        
                        try {
                            sh 'terraform apply -destroy -input=false tfplan-destroy'
                        } catch (Exception e) {
                            def errorMessage = e.getMessage()
                            echo 'Ignoring errors during destroy. Check the plan for issues.'
                            echo errorMessage
                            // Check for specific error messages and decide whether to skip destroy
                            if (errorMessage.contains('User with name aws-admin-vaut already exists') ||
                                errorMessage.contains('BucketAlreadyExists') ||
                                errorMessage.contains('Table already exists') ||
                                errorMessage.contains('resource record set already exists')) {
                                echo 'Resource already exists. Skipping destroy.'
                                currentBuild.result = 'SUCCESS'
                            } else {
                                error 'Unknown error occurred during destroy: ' + errorMessage
                            }
                        }
                    }
                }
              }
            }
        }
    }
}
