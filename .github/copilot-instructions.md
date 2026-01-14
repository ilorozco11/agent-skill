# E-commerce Recommendation System - Project Instructions

## Project Overview
Recommendation system for e-commerce, using machine learning to suggest products based on user behavior and product attributes.

## Tech Stack
- **Orchestration**: Apache Airflow 2.x on GCP Composer 2
- **Data Warehouse**: Google BigQuery
- **ML Platform**: BigQuery ML, Vertex AI
- **CI/CD**: Jenkins
- **Testing**: Pytest
- **Language**: Python 3.10+
- **Cloud**: Google Cloud Platform (GCP)

## Coding Conventions

### Python
- Follow PEP 8 style guide
- Use type hints for all functions
- Maximum line length: 120 characters
- Use `async/await` for async operations
- Docstrings: Google style format

### Naming Conventions
- DAGs: `{domain}_{action}_{frequency}` (e.g., `recommendation_train_daily`)
- Tasks: `{verb}_{noun}` (e.g., `extract_user_events`, `load_to_bigquery`)
- Tables: `{project}.{dataset}.{table_name}` with snake_case
- Models: `{model_type}_{version}` (e.g., `matrix_factorization_v1`)

### File Structure
```
src/
├── dags/                    # Airflow DAG definitions
│   ├── recommendation/      # Recommendation pipeline DAGs
│   └── utils/               # Shared DAG utilities
├── sql/                     # BigQuery SQL queries
│   ├── features/            # Feature engineering queries
│   └── models/              # ML model queries
├── tests/                   # Pytest test files
│   ├── unit/
│   └── integration/
└── config/                  # Configuration files
```

## Best Practices

### Airflow
- Always set `catchup=False` unless backfill is needed
- Use deferrable operators for long-running tasks
- Keep DAGs idempotent and atomic
- Use TaskFlow API for Python tasks

### BigQuery
- Partition tables by date for time-series data
- Use clustering for frequently filtered columns
- Avoid `SELECT *` - specify only needed columns
- Use `_TABLE_SUFFIX` for sharded tables

### Security
- Never hardcode credentials
- Use GCP Secret Manager for sensitive data
- Apply least privilege IAM roles
- Encrypt data at rest and in transit

## Do NOT
- Do not commit credentials or API keys
- Do not use deprecated Airflow operators
- Do not create DAGs with tight coupling
- Do not skip tests for data validation
