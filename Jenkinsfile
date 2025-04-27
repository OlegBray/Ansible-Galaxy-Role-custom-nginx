pipeline {
    agent any
    environment {
        VAULT_TOKEN = credentials('vault-token')
        VM_HOST = 'oleg.aws.cts.care'
        VM_USER = 'ubuntu'
    }
    stages {
        stage('Get SSH Private Key from Vault') {
            steps {
                script {
                    // Create a directory for storing credentials securely
                    sh 'mkdir -p ~/.ssh_temp && chmod 700 ~/.ssh_temp'
                    
                    // Retrieve private key from Vault using direct curl with basic shell processing
                    withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
                        sh '''
                            # Get response from Vault 
                            RESPONSE=$(curl --silent --header "X-Vault-Token: $VAULT_TOKEN" --request GET http://vault:8200/v1/secret/aws/privat-key)
                
                            # Extract the private key
                            echo "$RESPONSE" | sed 's/.*"value":"//' | sed 's/".*//' > ~/.ssh_temp/ssh_key.pem
                            sed -i 's/\\\\n/\\n/g' ~/.ssh_temp/ssh_key.pem
                            chmod 600 ~/.ssh_temp/ssh_key.pem
                        '''
                    }

                    // Dynamically get the EC2 instance IP using AWS CLI
                    // sh """
                    //     EC2_IP=$(aws ec2 describe-instances --region {{ aws_region }} --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
                    //     echo "EC2 IP: $EC2_IP"
                    // """
                    
                    
                    // echo "Private Key retrieved and stored securely"
                    
                    // // Test connection to VM
                    // sh """
                    //     ssh -o StrictHostKeyChecking=no -i ~/.ssh_temp/ssh_key.pem ${VM_USER}@${VM_HOST} 'echo "Successfully connected to the virtual machine!"'
                    // """
                }
            }
        }

        
        
        stage('Install Ansible') {
            steps {
                // Install ansible on Jenkins agent if not already installed
                sh '''
                    if ! command -v ansible-playbook &> /dev/null; then
                        sudo apt-get update
                        sudo apt-get install -y ansible
                    fi
                '''
            }
        }

        stage('Install Ansible AWS Collection') {
            steps {
                sh '''
                    echo "Installing amazon.aws and community.aws collection..."
                    ansible-galaxy collection install amazon.aws
                    ansible-galaxy collection install community.aws

                '''
            }
        }
        
        stage('Prepare Inventory') {
            steps {
                // Create inventory file for the target VM
                sh '''
                    echo "[webserver]" > inventory
                    echo "${VM_HOST} ansible_user=${VM_USER} ansible_ssh_private_key_file=~/.ssh_temp/ssh_key.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> inventory
                    cat inventory
                '''
            }
        }
        
        stage('Run Ansible Playbook') {
            steps {
                // Run ansible playbook from Jenkins against the remote VM
                sh '''
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i inventory nginx.yml -v --extra vars "random=`${random}`"
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                // Clean up the SSH key after use
                sh 'rm -rf ~/.ssh_temp'
                echo "Credentials cleaned up"
            }
        }
    }
    
    post {
        always {
            // Ensure credentials are always cleaned up, even if the pipeline fails
            sh 'rm -rf ~/.ssh_temp || true'
        }
    }
}

