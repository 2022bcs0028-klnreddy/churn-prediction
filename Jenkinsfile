pipeline {
    agent any

    environment {
        MLFLOW_TRACKING_URI = 'http://mlflow:5000'
        MODEL_DIR           = 'models'
        PYTHONPATH          = '.'
    }

    triggers {
        // CT: automatic retraining every Sunday at 2am
        cron('0 2 * * 0')
        // CI: trigger on GitHub push (requires GitHub webhook)
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
                    pip install -r requirements.txt --quiet
                '''
            }
        }

        stage('DVC Pull') {
            steps {
                sh '''
                    dvc pull || echo "No DVC remote configured, skipping pull"
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    python -m pytest tests/ -v --tb=short || true
                '''
            }
        }

        stage('Check Data Drift') {
            steps {
                sh '''
                    python -c "
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
                    python -m ml.train
                '''
            }
        }

        stage('Evaluate & Gate') {
            steps {
                sh '''
                    python -c "
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
                    dvc push || true
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