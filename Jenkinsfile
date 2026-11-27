pipeline {
    agent any

    options {
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'GitHub repository checkout completed'
                echo "Workspace: ${env.WORKSPACE}"
            }
        }

        stage('Validate Repository Files') {
            steps {
                sh '''
                    set -e

                    echo "Checking required files..."

                    test -f ansible/ansible.cfg
                    test -f ansible/inventory/inventory.ini
                    test -f ansible/playbooks/deploy-php.yml
                    test -f app/index.php
                    test -f sonar-project.properties

                    echo "All required files are present."
                '''
            }
        }

        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    set -e

                    echo "Starting Gitleaks secret scan..."

                    gitleaks git . --redact --exit-code 1

                    echo "Gitleaks scan completed successfully."
                '''
            }
        }

        stage('Ansible Version Check') {
            steps {
                sh '''
                    set -e

                    ansible --version
                    ansible-playbook --version
                '''
            }
        }

        stage('Ansible Syntax Validation') {
            steps {
                sh '''
                    set -e

                    cd ansible

                    export ANSIBLE_CONFIG="$PWD/ansible.cfg"

                    ansible-playbook \
                        -i inventory/inventory.ini \
                        playbooks/deploy-php.yml \
                        --syntax-check
                '''
            }
        }

        stage('SonarQube Code Quality Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy PHP Application') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'app-server-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -e

                        cd ansible

                        export ANSIBLE_CONFIG="$PWD/ansible.cfg"

                        chmod 600 "$SSH_KEY"

                        ansible-playbook \
                            -i inventory/inventory.ini \
                            playbooks/deploy-php.yml \
                            --private-key="$SSH_KEY" \
                            -e "ansible_user=$SSH_USER"
                    '''
                }
            }
        }

        stage('Deployment Completed') {
            steps {
                echo 'Ansible deployment stage completed successfully.'
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed. Review the Jenkins console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
