# hardening pipeline script for jenkins

pipeline {
    agent any

    parameters {
        string(name: 'TARGET_SERVER', defaultValue: '', description: 'Enter the server hostname or IP to deploy to')
    }

    environment {
        ANSIBLE_USER = 'richardp'
        INVENTORY = "${params.TARGET_SERVER},"
        SSH_CHECK_PLAYBOOK = 'ssh-check.yml'
        UPDATE_PLAYBOOK = 'hardening.yml'
        REPORT_DIR = 'reports'
        REPORT_FILE = 'report.html'
    }

    stages {
        stage('Validate Server Input') {
            steps {
                script {
                    if (!params.TARGET_SERVER?.trim()) {
                        error "No server specified. Please provide a TARGET_SERVER to proceed."
                    }
                    echo "Server provided: ${params.TARGET_SERVER}"
                }
            }
        }

        stage('Pre-check SSH Connectivity') {
            steps {
                echo "🔍 Running SSH connectivity check using Ansible playbook"
                sh """
                    ansible-playbook -i ${env.INVENTORY} -u ${env.ANSIBLE_USER} ${env.SSH_CHECK_PLAYBOOK}
                """
            }
        }

        stage('Run Updates') {
            steps {
                echo "🚀 Running update playbook"
                sh """
                    mkdir -p ${env.REPORT_DIR}
                    ansible-playbook -i ${env.INVENTORY} -u ${env.ANSIBLE_USER} ${env.UPDATE_PLAYBOOK} \
                    --extra-vars="code_user=${env.ANSIBLE_USER} build_number=${env.BUILD_NUMBER} build_date=\$(date +%Y-%m-%d) build_status=Success"
                """
                sh """
                    mkdir -p ${env.REPORT_DIR}
                    echo "<html><body><h1>Hardening Report</h1><p>Build ${env.BUILD_NUMBER} completed.</p></body></html>" \
                        > ${env.REPORT_DIR}/${env.REPORT_FILE}

                    ansible-playbook -i ${env.INVENTORY} -u ${env.ANSIBLE_USER} ${env.UPDATE_PLAYBOOK} \
                    --extra-vars="code_user=${env.ANSIBLE_USER} build_number=${env.BUILD_NUMBER} build_date=\$(date +%Y-%m-%d) build_status=Success"
                   """

            }
        }

        stage('Publish Report') {
            steps {
                publishHTML(target: [
                    reportName: 'Update Report',
                    reportDir: "${env.REPORT_DIR}",
                    reportFiles: "${env.REPORT_FILE}",
                    keepAll: true
                ])
            }
        }
    }
}
