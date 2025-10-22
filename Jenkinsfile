pipeline {
    agent any

    environment {
        ANSIBLE_PLAYBOOK = "site.yml"
        ANSIBLE_INVENTORY = "postgresql_manager/tests/inventory"
        EMAIL_RECIPIENTS = "rahul85955r@gmail.com"   // Change to your team emails
        SLACK_CHANNEL = "#jenkins-notify"          // Change to your Slack channel
        VENV_PATH = "${WORKSPACE}/molecule-venv"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📦 Checking out code from GitHub..."
                checkout scm
            }
        }

        stage('Setup Environment') {
            steps {
                echo "⚙️ Setting up virtual environment..."
                sh """
                if [ ! -d ${VENV_PATH} ]; then
                    python3 -m venv ${VENV_PATH}
                fi
                . ${VENV_PATH}/bin/activate
                pip install --upgrade pip
                pip install ansible molecule ansible-lint docker
                """
            }
        }

        stage('Ansible Lint Check') {
            steps {
                echo "🧹 Running Ansible Lint..."
                sh """
                . ${VENV_PATH}/bin/activate
                ansible-lint ${ANSIBLE_PLAYBOOK}
                """
            }
        }

        stage('Molecule Role Test') {
            steps {
                echo "🧩 Running Molecule tests..."
                sh """
                . ${VENV_PATH}/bin/activate
                cd ${WORKSPACE}/ansible-postgresql-setup/postgresql_manager
                molecule test
                """
            }
        }

        stage('Dry Run (Test Mode)') {
            steps {
                echo "🧪 Running dry-run test..."
                sh """
                . ${VENV_PATH}/bin/activate
                ansible-playbook ${ANSIBLE_PLAYBOOK} -i ${ANSIBLE_INVENTORY} --check
                """
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                echo "🚀 Deploying PostgreSQL..."
                sh """
                . ${VENV_PATH}/bin/activate
                ansible-playbook ${ANSIBLE_PLAYBOOK} -i ${ANSIBLE_INVENTORY}
                """
            }
        }

        stage('PostgreSQL Health Check') {
            steps {
                echo "💚 Checking PostgreSQL service status..."
                sh """
                ansible all -i ${ANSIBLE_INVENTORY} -m shell -a 'systemctl status postgresql | grep active'
                """
            }
        }

        stage('Database Connectivity Test') {
            steps {
                echo "🔗 Testing database connection..."
                sh """
                ansible all -i ${ANSIBLE_INVENTORY} -m shell -a 'psql -U postgres -c "SELECT version();"'
                """
            }
        }
    }

    post {
        always {
            echo "📊 Generating Deployment Report..."
            publishHTML(target: [
                reportDir: 'reports',
                reportFiles: 'index.html',
                reportName: 'Ansible Deployment Report',
                keepAll: true
            ])
        }

        success {
            echo "✅ PostgreSQL successfully deployed!"
            
            mail(to: "${EMAIL_RECIPIENTS}",
                 subject: "Jenkins Job Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                 body: "✅ PostgreSQL deployment completed successfully!\nCheck report: ${env.BUILD_URL}")
            
            slackSend(channel: "${SLACK_CHANNEL}",
                      color: 'good',
                      message: "✅ PostgreSQL deployed successfully! Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }

        failure {
            echo "❌ Deployment failed. Check logs!"
            
            mail(to: "${EMAIL_RECIPIENTS}",
                 subject: "Jenkins Job Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                 body: "❌ PostgreSQL deployment failed!\nCheck console output: ${env.BUILD_URL}")
            
            slackSend(channel: "${SLACK_CHANNEL}",
                      color: 'danger',
                      message: "❌ Deployment failed! Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }
    }
}
