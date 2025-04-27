pipeline {
    agent any
    environment {
        VAULT_ADDR = 'http://vault:8200'
        AWS_CREDS_PATH = 'secret/data/aws/creds'
    }
    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    # Check if Python is installed
                    echo "Installing Python 3..."
                    sudo apt update -y || echo "Warning: apt-get update failed"
                    sudo apt install -y python3 python3-pip || echo "Warning: Python installation failed"
                    
                    # Check if pip is installed
                    echo "Installing pip3..."
                    sudo apt install -y python3-pip || echo "Warning: pip installation failed"
                    
                    # Install Ansible (try both with and without sudo)
                    echo "Installing Ansible..."
                    sudo apt install ansible || echo "Warning: Ansible installation failed"
                    
                    # Display versions
                    python3 --version || echo "Python not installed"
                    pip3 --version || echo "pip not installed"
                    ansible --version || echo "Ansible not installed"
                '''
            }
        }
        
        stage('Read AWS Credentials from Vault') {
            steps {
                withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
                    script {
                        try {
                            def response = httpRequest(
                                httpMode: 'GET',
                                url: "${env.VAULT_ADDR}/v1/${env.AWS_CREDS_PATH}",
                                customHeaders: [[name: 'X-Vault-Token', value: VAULT_TOKEN]],
                                validResponseCodes: '200'
                            )
                            
                            echo "Vault response status: ${response.status}"
                            
                            def json = readJSON text: response.content
                            
                            if (json.data && json.data.data && json.data.data.access_key_id && json.data.data.secret_access_key) {
                                env.AWS_ACCESS_KEY_ID = json.data.data.access_key_id
                                env.AWS_SECRET_ACCESS_KEY = json.data.data.secret_access_key
                                echo "✅ AWS credentials retrieved successfully"
                            } else {
                                error "AWS credentials not found in Vault response!"
                            }
                        } catch (Exception e) {
                            echo "Error accessing Vault: ${e.message}"
                            throw e
                        }
                    }
                }
            }
        }

        stage('Install Ansible') {
            steps {
                // Install ansible on Jenkins agent if not already installed
                sh '''
                    if ! command -v ansible-playbook &> /dev/null; then
                        sudo apt update
                        sudo apt install -y ansible
                    fi
                '''
            }
        }
        
        stage('Run Ansible Playbook') {
            steps {
                withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", 
                         "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}"]) {
                    sh '''
                        # Add ~/.local/bin to PATH if it exists
                        export PATH=$PATH:~/.local/bin
                        
                        # Check for ansible-playbook
                        which ansible-playbook || echo "Warning: ansible-playbook not found in PATH"
                        
                        # Run ansible-playbook
                        ansible-playbook -i inventory.ini nginx.yml -v || echo "Warning: Ansible playbook execution failed"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY'
        }
    }
}