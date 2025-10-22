pipeline {
    agent any

    environment {
        ANSIBLE_PLAYBOOK = "site.yml"
        ANSIBLE_INVENTORY = "postgresql_manager/tests/inventory"
        EMAIL_RECIPIENTS = "rahul85955r@gmail.com"
        SLACK_CHANNEL = "#jenkins-notify"
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
                sh '''
                    if [ ! -d "$HOME/molecule-venv" ]; then
                        python3 -m venv ~/molecule-venv
                        source ~/molecule-venv/bin/activate
                        pip install --upgrade pip
                        pip install molecule ansible ansible-lint docker
                    else
                        echo "✅ Virtualenv already exists."
                    fi
                '''
            }
        }

        stage('Ansible Lint Check') {
            steps {
                echo "🧹 Running Ansible Lint..."
                sh '''
                    source ~/molecule-venv/bin/activate
                    ansible-lint ${ANSIBLE_PLAYBOOK}
                '''
            }
        }

        stage('Molecule Test') {
            steps {
                echo "🚀 Running Molecule test..."
                sh '''
                    source ~/molecule-venv/bin/activate
                    cd ~/ansible-postgresql-setup/postgresql_manager
                    molecule test
                '''
            }
        }

        stage('Dry Run (Test Mode)') {
            steps {
                echo "🧪 Running dry-run test..."
                sh '''
                    source ~/molecule-venv/bin/activate
                    ansible-playbook ${ANSIBLE_PLAYBOOK} -i ${ANSIBLE_INVENTORY} --check
                '''
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                echo "🚀 Deploying PostgreSQL..."
                sh '''
                    source ~/molecule-venv/bin/activate
                    ansible-playbook ${ANSIBLE_PLAYBOOK} -i ${ANSIBLE_INVENTORY}
                '''
            }
        }

        stage('PostgreSQL Health Check') {
            steps {
                echo "💚 Checking PostgreSQL service status..."
                sh '''
                    ansible all -i ${ANSIBLE_INVENTORY} -m shell -a 'systemctl status postgresql | grep active'
                '''
            }
        }

        stage('Database Connectivity Test') {
            steps {
                echo "🔗 Testing database connection..."
                sh '''
                    ansible all -i ${ANSIBLE_INVENTORY} -m shell -a 'psql -U postgres -c "SELECT version();"'
                '''
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
                 subject: "✅ Jenkins Job Success: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                 body: "PostgreSQL deployment completed successfully!\nCheck report: ${env.BUILD_URL}")
            slackSend(channel: "${SLACK_CHANNEL}", color: 'good',
                      message: "✅ PostgreSQL deployed successfully! Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }

        failure {
            echo "❌ Deployment failed. Check logs!"
            mail(to: "${EMAIL_RECIPIENTS}",
                 subject: "❌ Jenkins Job Failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                 body: "PostgreSQL deployment failed!\nCheck console output: ${env.BUILD_URL}")
            slackSend(channel: "${SLACK_CHANNEL}", color: 'danger',
                      message: "❌ Deployment failed! Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER}")
        }
    }
}
