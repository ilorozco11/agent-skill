---
name: Jenkins CI/CD Pipeline
description: Support for creating Jenkins pipelines for data projects and Airflow DAG deployment
---

# Jenkins CI/CD Pipeline Skill

This skill guides Copilot to create Jenkins pipelines for data engineering projects.

## When to Use
- Create Jenkinsfile for data projects
- Setup CI/CD for Airflow DAGs
- Configure automated testing
- Deploy to GCP Composer

## Jenkinsfile Template

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: python
                    image: python:3.10-slim
                    command: ['sleep', '99d']
                  - name: gcloud
                    image: google/cloud-sdk:latest
                    command: ['sleep', '99d']
            '''
        }
    }
    
    environment {
        GCP_PROJECT = credentials('gcp-project-id')
        COMPOSER_ENV = 'recommendation-prod'
        COMPOSER_REGION = 'asia-southeast1'
        SLACK_WEBHOOK = credentials('slack-webhook')
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                container('python') {
                    sh '''
                        pip install -r requirements.txt
                        pip install -r requirements-dev.txt
                    '''
                }
            }
        }
        
        stage('Lint') {
            parallel {
                stage('Ruff') {
                    steps {
                        container('python') {
                            sh 'ruff check src/'
                        }
                    }
                }
                stage('Black') {
                    steps {
                        container('python') {
                            sh 'black --check src/'
                        }
                    }
                }
                stage('Mypy') {
                    steps {
                        container('python') {
                            sh 'mypy src/'
                        }
                    }
                }
            }
        }
        
        stage('DAG Validation') {
            steps {
                container('python') {
                    sh '''
                        # Validate DAG syntax
                        python -c "
import sys
from airflow.models import DagBag
dag_bag = DagBag(dag_folder='src/dags', include_examples=False)
if dag_bag.import_errors:
    for dag_id, error in dag_bag.import_errors.items():
        print(f'DAG {dag_id}: {error}')
    sys.exit(1)
print(f'Validated {len(dag_bag.dags)} DAGs successfully')
"
                    '''
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                container('python') {
                    sh '''
                        pytest tests/unit \
                            --cov=src \
                            --cov-report=xml \
                            --cov-report=html \
                            --junitxml=test-results.xml \
                            -v
                    '''
                }
            }
            post {
                always {
                    junit 'test-results.xml'
                    publishHTML(target: [
                        reportName: 'Coverage Report',
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html'
                    ])
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                container('python') {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh '''
                            pytest tests/integration \
                                --junitxml=integration-test-results.xml \
                                -v
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                container('gcloud') {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh '''
                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                            gcloud config set project $GCP_PROJECT
                            
                            # Sync DAGs to Composer
                            DAGS_BUCKET=$(gcloud composer environments describe recommendation-staging \
                                --location $COMPOSER_REGION \
                                --format='value(config.dagGcsPrefix)')
                            
                            gsutil -m rsync -r -d src/dags/ $DAGS_BUCKET/
                            gsutil -m rsync -r src/sql/ $DAGS_BUCKET/../data/sql/
                        '''
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                container('gcloud') {
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh '''
                            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                            gcloud config set project $GCP_PROJECT
                            
                            # Sync DAGs to Production Composer
                            DAGS_BUCKET=$(gcloud composer environments describe $COMPOSER_ENV \
                                --location $COMPOSER_REGION \
                                --format='value(config.dagGcsPrefix)')
                            
                            gsutil -m rsync -r -d src/dags/ $DAGS_BUCKET/
                            gsutil -m rsync -r src/sql/ $DAGS_BUCKET/../data/sql/
                            
                            echo "Deployed to $DAGS_BUCKET"
                        '''
                    }
                }
            }
        }
    }
    
    post {
        success {
            slackSend(
                channel: '#data-pipeline-ci',
                color: 'good',
                message: "✅ Pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#data-pipeline-ci',
                color: 'danger',
                message: "❌ Pipeline failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
            )
        }
        always {
            cleanWs()
        }
    }
}
```

## DAG Validation Script

```python
# scripts/validate_dags.py
#!/usr/bin/env python3
"""Validate Airflow DAGs before deployment."""

import sys
from pathlib import Path
from airflow.models import DagBag

def validate_dags(dag_folder: str) -> bool:
    """Validate all DAGs in the specified folder."""
    dag_bag = DagBag(dag_folder=dag_folder, include_examples=False)
    
    errors = []
    
    # Check for import errors
    if dag_bag.import_errors:
        for dag_id, error in dag_bag.import_errors.items():
            errors.append(f"Import error in {dag_id}: {error}")
    
    # Validate DAG configurations
    for dag_id, dag in dag_bag.dags.items():
        # Check for required tags
        if not dag.tags:
            errors.append(f"{dag_id}: Missing tags")
        
        # Check for owner
        if dag.default_args.get("owner") == "airflow":
            errors.append(f"{dag_id}: Default owner 'airflow' should be changed")
        
        # Check for catchup disabled
        if dag.catchup:
            errors.append(f"{dag_id}: catchup should be False")
    
    if errors:
        print("Validation errors found:")
        for error in errors:
            print(f"  ❌ {error}")
        return False
    
    print(f"✅ Validated {len(dag_bag.dags)} DAGs successfully")
    return True

if __name__ == "__main__":
    dag_folder = sys.argv[1] if len(sys.argv) > 1 else "src/dags"
    success = validate_dags(dag_folder)
    sys.exit(0 if success else 1)
```

## Advanced Deployment Strategies

### Canary Deployment
```groovy
stage('Canary Deployment') {
    when { branch 'main' }
    steps {
        script {
            container('gcloud') {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS

                        # Deploy to canary folder first
                        DAGS_BUCKET=$(gcloud composer environments describe $COMPOSER_ENV \
                            --location $COMPOSER_REGION \
                            --format='value(config.dagGcsPrefix)')

                        gsutil -m rsync -r src/dags/ ${DAGS_BUCKET}/canary/

                        echo "Canary deployed. Monitoring for 30 minutes..."
                        sleep 1800

                        # Check error rate in canary
                        ERROR_COUNT=$(gcloud logging read "
                            resource.type=cloud_composer_environment
                            AND severity>=ERROR
                            AND labels.dag_folder=canary
                            AND timestamp>=\"$(date -u -d '30 minutes ago' '+%Y-%m-%dT%H:%M:%SZ')\"
                        " --limit=1000 --format=json | jq '. | length')

                        if [ $ERROR_COUNT -gt 5 ]; then
                            echo "Canary deployment failed with $ERROR_COUNT errors"
                            exit 1
                        fi

                        # Promote to production
                        echo "Canary successful. Promoting to production..."
                        gsutil -m rsync -r -d ${DAGS_BUCKET}/canary/ ${DAGS_BUCKET}/dags/
                    '''
                }
            }
        }
    }
}
```

### Blue-Green Deployment
```groovy
stage('Blue-Green Deploy') {
    steps {
        script {
            def targetEnv = env.ACTIVE_ENV == 'blue' ? 'green' : 'blue'

            echo "Deploying to ${targetEnv} environment"

            container('gcloud') {
                sh """
                    # Deploy to inactive environment
                    DAGS_BUCKET=\$(gcloud composer environments describe recommendation-${targetEnv} \
                        --location ${COMPOSER_REGION} \
                        --format='value(config.dagGcsPrefix)')

                    gsutil -m rsync -r -d src/dags/ \${DAGS_BUCKET}/

                    # Run smoke tests
                    python scripts/smoke_test.py --environment ${targetEnv}
                """
            }

            // Manual verification
            input message: "Switch traffic to ${targetEnv}?", ok: 'Switch'

            // Update DNS or load balancer to point to new environment
            sh "gcloud dns record-sets update airflow.example.com --rrdatas=\${NEW_ENV_IP}"
        }
    }
}
```

### Feature Flags for Gradual Rollout
```python
# scripts/feature_flags.py
from typing import Dict
import hashlib

class FeatureFlags:
    """Manage feature flags for gradual rollout."""

    def __init__(self, config: Dict[str, float]):
        self.config = config  # feature_name -> rollout_percentage

    def is_enabled(self, feature: str, dag_id: str) -> bool:
        """Check if feature is enabled for this DAG."""
        if feature not in self.config:
            return False

        rollout_pct = self.config[feature]
        if rollout_pct >= 100:
            return True
        if rollout_pct <= 0:
            return False

        # Deterministic hash-based rollout
        hash_val = int(hashlib.md5(f"{feature}:{dag_id}".encode()).hexdigest(), 16)
        return (hash_val % 100) < rollout_pct

# Usage in DAG
# if feature_flags.is_enabled('new_algorithm', dag_id):
#     use_new_algorithm()
```

## Security Scanning

### SAST with Bandit
```groovy
stage('Security Scan') {
    parallel {
        stage('SAST - Bandit') {
            steps {
                container('python') {
                    sh '''
                        pip install bandit
                        bandit -r src/ -f json -o bandit-report.json || true
                    '''
                    recordIssues(
                        tools: [bandit(pattern: 'bandit-report.json')],
                        qualityGates: [[threshold: 1, type: 'TOTAL_HIGH', unstable: false]]
                    )
                }
            }
        }

        stage('Dependency Scan - Safety') {
            steps {
                container('python') {
                    sh '''
                        pip install safety
                        safety check --json --output safety-report.json || true
                    '''
                }
            }
        }

        stage('Secret Detection') {
            steps {
                container('python') {
                    sh '''
                        pip install detect-secrets
                        detect-secrets scan --all-files > secrets-baseline.json

                        SECRETS_COUNT=$(jq '.results | length' secrets-baseline.json)
                        if [ $SECRETS_COUNT -gt 0 ]; then
                            echo "ERROR: Secrets detected in code!"
                            jq '.results' secrets-baseline.json
                            exit 1
                        fi
                    '''
                }
            }
        }
    }
}
```

### SQL Injection Detection
```groovy
stage('SQL Security Check') {
    steps {
        container('python') {
            sh '''
                # Check for SQL injection vulnerabilities
                python scripts/check_sql_security.py src/sql/
            '''
        }
    }
}
```

```python
# scripts/check_sql_security.py
import re
import sys
from pathlib import Path

def check_sql_file(filepath: Path) -> list:
    """Check SQL file for security issues."""
    issues = []
    content = filepath.read_text()

    # Check for string concatenation (potential SQL injection)
    if re.search(r'\+\s*["\'].*["\']', content):
        issues.append(f"{filepath}: Potential SQL injection via string concatenation")

    # Check for format strings without parameterization
    if re.search(r'\.format\(', content) or re.search(r'%\s*\(', content):
        issues.append(f"{filepath}: Use parameterized queries instead of format()")

    return issues

if __name__ == "__main__":
    sql_dir = Path(sys.argv[1])
    all_issues = []

    for sql_file in sql_dir.rglob("*.sql"):
        all_issues.extend(check_sql_file(sql_file))

    if all_issues:
        for issue in all_issues:
            print(f"❌ {issue}")
        sys.exit(1)

    print("✅ No SQL security issues found")
```

## Performance Testing

### Query Performance Benchmarking
```python
# scripts/performance_test.py
import json
import time
from google.cloud import bigquery
from typing import Dict

def benchmark_queries(baseline_file: str, max_regression_pct: float = 10.0) -> bool:
    """Benchmark query performance against baseline."""

    client = bigquery.Client()

    queries = {
        "user_features": "SELECT * FROM `project.recommendation.user_features` LIMIT 1000",
        "product_stats": "SELECT * FROM `project.recommendation.product_stats` LIMIT 1000",
    }

    results = {}
    for query_name, query_sql in queries.items():
        # Dry run for cost
        job_config = bigquery.QueryJobConfig(dry_run=True)
        dry_run = client.query(query_sql, job_config=job_config)

        # Actual run for performance
        start = time.time()
        query_job = client.query(query_sql)
        query_job.result()
        duration_ms = (time.time() - start) * 1000

        results[query_name] = {
            "duration_ms": duration_ms,
            "bytes_processed": dry_run.total_bytes_processed,
        }

    # Compare with baseline
    try:
        with open(baseline_file, 'r') as f:
            baseline = json.load(f)

        regressions = []
        for name, current in results.items():
            if name in baseline:
                baseline_duration = baseline[name]["duration_ms"]
                regression_pct = ((current["duration_ms"] - baseline_duration) / baseline_duration) * 100

                if regression_pct > max_regression_pct:
                    regressions.append(f"{name}: {regression_pct:.1f}% slower")

        if regressions:
            print("Performance regressions:")
            for r in regressions:
                print(f"  ❌ {r}")
            return False
    except FileNotFoundError:
        print("No baseline found. Creating new baseline...")

    # Update baseline
    with open(baseline_file, 'w') as f:
        json.dump(results, f, indent=2)

    return True
```

```groovy
stage('Performance Tests') {
    steps {
        container('python') {
            withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                sh '''
                    python scripts/performance_test.py \
                        --baseline performance-baseline.json \
                        --max-regression-percent 10
                '''
            }
        }
    }
}
```

## Artifact Management

### DAG Versioning
```python
# scripts/version_dags.py
import hashlib
import json
from pathlib import Path
from datetime import datetime

def calculate_dag_hash(dag_file: Path) -> str:
    """Calculate hash for DAG file."""
    return hashlib.sha256(dag_file.read_bytes()).hexdigest()[:12]

def create_version_manifest(dags_dir: Path, output_file: Path):
    """Create version manifest for rollback."""
    manifest = {
        "version": datetime.now().isoformat(),
        "commit": os.getenv("GIT_COMMIT", "unknown"),
        "dags": {}
    }

    for dag_file in dags_dir.rglob("*.py"):
        if dag_file.name.startswith("test_"):
            continue

        relative_path = str(dag_file.relative_to(dags_dir))
        manifest["dags"][relative_path] = {
            "hash": calculate_dag_hash(dag_file),
            "size": dag_file.stat().st_size,
            "modified": datetime.fromtimestamp(dag_file.stat().st_mtime).isoformat()
        }

    output_file.write_text(json.dumps(manifest, indent=2))
    return manifest
```

```groovy
stage('Version and Archive') {
    steps {
        container('python') {
            sh '''
                # Create version manifest
                python scripts/version_dags.py \
                    --dags-dir src/dags \
                    --output version-manifest.json

                # Archive to GCS
                gsutil cp version-manifest.json \
                    gs://dag-versions/${GIT_COMMIT}/manifest.json

                # Archive DAGs
                tar -czf dags-${GIT_COMMIT}.tar.gz src/dags/
                gsutil cp dags-${GIT_COMMIT}.tar.gz \
                    gs://dag-versions/${GIT_COMMIT}/
            '''
        }
    }
}
```

### Quick Rollback
```bash
#!/bin/bash
# scripts/rollback.sh

PREVIOUS_VERSION=$1
COMPOSER_ENV="recommendation-prod"
REGION="asia-southeast1"

echo "Rolling back to version: $PREVIOUS_VERSION"

# Download previous version
gsutil cp gs://dag-versions/${PREVIOUS_VERSION}/dags-${PREVIOUS_VERSION}.tar.gz /tmp/
tar -xzf /tmp/dags-${PREVIOUS_VERSION}.tar.gz -C /tmp/

# Deploy to Composer
DAGS_BUCKET=$(gcloud composer environments describe $COMPOSER_ENV \
    --location $REGION \
    --format='value(config.dagGcsPrefix)')

gsutil -m rsync -r -d /tmp/src/dags/ ${DAGS_BUCKET}/

echo "Rollback completed"
```

## Pipeline Observability

### Build Metrics Collection
```groovy
post {
    always {
        script {
            def buildDuration = currentBuild.duration
            def buildResult = currentBuild.result ?: 'SUCCESS'

            // Send metrics to monitoring system
            sh """
                curl -X POST https://monitoring.example.com/api/metrics \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "pipeline": "${env.JOB_NAME}",
                        "build_number": ${env.BUILD_NUMBER},
                        "duration_ms": ${buildDuration},
                        "result": "${buildResult}",
                        "branch": "${env.BRANCH_NAME}",
                        "commit": "${env.GIT_COMMIT}",
                        "timestamp": "${new Date().format('yyyy-MM-dd HH:mm:ss')}"
                    }'
            """
        }
    }
}
```

### DORA Metrics Tracking
```python
# scripts/track_dora_metrics.py
from datetime import datetime, timedelta
from typing import Dict
import requests

class DORAMetrics:
    """Track DORA (DevOps Research and Assessment) metrics."""

    def __init__(self, api_url: str):
        self.api_url = api_url

    def track_deployment(self, success: bool):
        """Track deployment frequency and success rate."""
        requests.post(f"{self.api_url}/deployments", json={
            "timestamp": datetime.now().isoformat(),
            "success": success,
            "environment": "production"
        })

    def track_lead_time(self, commit_time: datetime, deploy_time: datetime):
        """Track lead time for changes."""
        lead_time_minutes = (deploy_time - commit_time).total_seconds() / 60
        requests.post(f"{self.api_url}/lead_time", json={
            "lead_time_minutes": lead_time_minutes,
            "timestamp": deploy_time.isoformat()
        })

    def track_mttr(self, incident_start: datetime, resolution_time: datetime):
        """Track Mean Time To Recovery."""
        mttr_minutes = (resolution_time - incident_start).total_seconds() / 60
        requests.post(f"{self.api_url}/mttr", json={
            "mttr_minutes": mttr_minutes,
            "timestamp": resolution_time.isoformat()
        })
```

## Best Practices
- Run DAG validation before deployment
- Use parallel stages for faster execution
- Require manual approval for production deploys
- Send notifications to Slack/Teams on completion
- Clean workspace after each build
- Use Kubernetes agents for scalability
- Implement security scanning in every pipeline
- Use canary or blue-green deployments for critical changes
- Track DORA metrics for continuous improvement
- Version and archive DAGs for quick rollback
- Run performance regression tests
- Detect secrets and vulnerabilities before deployment
- Maintain deployment frequency and lead time metrics
- Store deployment artifacts in versioned storage
- Implement automated rollback on failure detection
