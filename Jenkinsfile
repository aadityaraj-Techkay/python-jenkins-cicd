pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub'
            }
        }
        stage('Install dependencies and test') {
            steps {
                sh '''
                    python3 --version
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    python -m py_compile app.py
                    python -c "from app import app; c=app.test_client(); assert c.get('/health').status_code == 200"
                '''
            }
        }
        stage('Deploy to EC2') {
            steps {
                sh '''
                    rsync -a --delete --exclude='.git/' --exclude='.venv/' ./ /opt/python-task-manager/
                    /opt/python-task-manager/venv/bin/pip install -r /opt/python-task-manager/requirements.txt
                    sudo /usr/bin/systemctl restart python-task-manager
                    sudo /usr/bin/systemctl is-active --quiet python-task-manager
                '''
            }
        }
    }
    post {
        success { echo 'Build and deployment completed successfully!' }
        failure { echo 'Pipeline failed. Check the stage logs above.' }
    }
}
