pipeline {
    agent any

    environment {
        MLFLOW_TRACKING_URI = 'http://mlflow:5000'
        MODEL_DIR           = 'models'
        PYTHONPATH          = '.'
        VENV                = 'venv/bin'
    }

    triggers {
        cron('0 2 * * 0')
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/2022bcs0028-klnreddy/churn-prediction.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    venv/bin/pip install --upgrade pip --quiet
                    venv/bin/pip install -r requirements.txt --quiet
                '''
            }
        }

        stage('DVC Pull') {
            steps {
                sh '''
                    venv/bin/dvc pull || echo "No DVC remote configured, skipping pull"
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    venv/bin/python -m pytest tests/ -v --tb=short || true
                '''
            }
        }

        stage('Check Data Drift') {
            steps {
                sh '''
                    venv/bin/python -c "
from mlops.drift.drift_detection import check_drift
import sys
drift = check_drift()
print(f'Drift detected: {drift}')
"
                '''
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                    venv/bin/python -m ml.train
                '''
            }
        }

        stage('Evaluate & Gate') {
            steps {
                sh '''
                    venv/bin/python -c "
import json, sys
with open('metrics.json') as f:
    m = json.load(f)
print(f'F1={m[\"f1\"]:.4f}  ROC-AUC={m[\"roc_auc\"]:.4f}')
if m['f1'] < 0.70:
    print('F1 below threshold — blocking deployment')
    sys.exit(1)
print('Model passed quality gate')
"
                '''
            }
        }

        stage('Build & Deploy Docker') {
            steps {
                sh '''
                    docker-compose build ml-api
                    docker-compose up -d ml-api
                '''
            }
        }

        stage('Push Metrics') {
            steps {
                sh '''
                    git config user.email "jenkins@churn-prediction"
                    git config user.name "Jenkins"
                    git add metrics.json || true
                    git commit -m "CI: update metrics [skip ci]" || true
                    git push origin main || true
                    venv/bin/dvc push || true
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline succeeded — new model is live!'
        }
        failure {
            echo 'Pipeline failed — check logs above.'
        }
    }
}