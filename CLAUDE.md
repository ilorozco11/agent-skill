# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains **GitHub Copilot Agent Skills** for developing an e-commerce recommendation system. It provides structured guidance to GitHub Copilot for building ML-powered data pipelines using Apache Airflow on GCP Composer 2, BigQuery for data warehousing and ML, and Jenkins for CI/CD.

The project is a **documentation-only repository** - it contains skill definitions and examples rather than actual production code. The skills guide Copilot to follow specific patterns and best practices when generating code for the recommendation system.

## Tech Stack

- **Orchestration**: Apache Airflow 2.x on GCP Composer 2
- **Data Warehouse**: Google BigQuery
- **ML Platform**: BigQuery ML, Vertex AI
- **CI/CD**: Jenkins
- **Testing**: Pytest
- **Language**: Python 3.10+
- **Cloud**: Google Cloud Platform (GCP)

## Repository Structure

```
.github/
├── copilot-instructions.md              # Project-wide Copilot instructions
└── skills/
    ├── airflow-dag/SKILL.md             # DAG development patterns
    ├── bigquery/SKILL.md                # Query & feature engineering
    ├── composer/SKILL.md                # GCP Composer deployment
    ├── jenkins-cicd/SKILL.md            # CI/CD pipelines
    ├── pytest-testing/SKILL.md          # Testing data pipelines
    └── recommendation-ml/SKILL.md       # ML model development
```

## Key Architectural Patterns

### Naming Conventions

Follow these conventions strictly as defined in `.github/copilot-instructions.md`:

- **DAGs**: `{domain}_{action}_{frequency}` (e.g., `recommendation_train_daily`)
- **Tasks**: `{verb}_{noun}` (e.g., `extract_user_events`, `load_to_bigquery`)
- **BigQuery Tables**: `{project}.{dataset}.{table_name}` with snake_case
- **ML Models**: `{model_type}_{version}` (e.g., `matrix_factorization_v1`)

### Recommendation System Architecture

The system follows a layered architecture:

1. **Data Sources** → User events, product catalog, transactions, user profiles
2. **BigQuery Data Warehouse** → Feature engineering (user, product, interaction features)
3. **BigQuery ML Models** → Matrix Factorization (collaborative filtering) or Wide-and-Deep (hybrid)
4. **Serving Layer** → Real-time API (Vertex AI) or batch recommendations (BigQuery → Redis/Firestore)

### Airflow DAG Patterns

- Use **TaskFlow API** (`@dag`, `@task` decorators) for new DAGs
- Always set `catchup=False` unless backfill is explicitly needed
- Use **deferrable operators** for long-running tasks to release workers
- Set `execution_timeout` on all tasks to prevent hanging
- Keep DAGs **idempotent** and **atomic**

### BigQuery Best Practices

- **Always partition tables** by date for time-series data
- **Use clustering** on frequently filtered columns (user_id, product_id)
- **Specify columns explicitly** - avoid `SELECT *`
- **Use `SAFE_DIVIDE`** to prevent division by zero errors
- **Use materialized views** for expensive, frequently-accessed queries

### Testing Strategy

Tests are organized into:
- **Unit tests** (`tests/unit/`): Mock external services (BigQuery, GCS), test DAG structure and task logic
- **Integration tests** (`tests/integration/`): Require GCP credentials, validate data quality in BigQuery
- **DAG validation**: Check structure, tags, default args, catchup settings before deployment

Use pytest markers: `@pytest.mark.unit` and `@pytest.mark.integration`

## Development Workflow

### Working with Skills

This repository contains GitHub Copilot agent skills. When using GitHub Copilot in VS Code:

1. Enable agent skills in VS Code Settings: `chat.useAgentSkills` → `true`
2. Enable instruction files: `github.copilot.chat.codeGeneration.useInstructionFiles` → `true`
3. Skills automatically guide Copilot when generating code for:
   - Airflow DAGs
   - BigQuery queries
   - ML models
   - Tests
   - CI/CD pipelines
   - GCP Composer configuration

### CI/CD Pipeline Flow

Jenkins pipeline stages:
1. **Checkout** → Pull code
2. **Install Dependencies** → Python packages
3. **Lint** → Ruff, Black, Mypy (parallel)
4. **DAG Validation** → Check DAG syntax with `DagBag`
5. **Unit Tests** → Pytest with coverage
6. **Integration Tests** → Only on `main`/`develop` branches
7. **Deploy to Staging** → On `develop` branch
8. **Deploy to Production** → On `main` branch with manual approval

### Deployment to GCP Composer

DAGs are deployed via:
```bash
gsutil -m rsync -r -d src/dags/ gs://{composer-bucket}/dags/
gsutil -m rsync -r src/sql/ gs://{composer-bucket}/data/sql/
```

## Important Conventions from `.github/copilot-instructions.md`

### Python Style
- Follow PEP 8
- Use type hints for all functions
- Maximum line length: 120 characters
- Docstrings: Google style format

### Security Rules
- Never hardcode credentials or API keys
- Use GCP Secret Manager for sensitive data
- Apply least privilege IAM roles
- Use Airflow's Secret Manager backend: `CloudSecretManagerBackend`

### Airflow-Specific Rules
- Do NOT use deprecated operators
- Do NOT create tightly coupled DAGs
- Do NOT skip data validation tests
- Always use `email_on_failure: True` in default args
- Set `retries >= 3` with appropriate `retry_delay`

## ML Model Development

### Feature Engineering Patterns

1. **User Features**: Recency (days since last visit), Frequency (visit days, events), Monetary (total spent, purchase count)
2. **Product Features**: Popularity metrics (viewers, cart adds, purchases), conversion rates, ratings
3. **Interaction Features**: Weighted scores (view=1, add_to_cart=3, purchase=5)

### Model Types

- **Collaborative Filtering**: Matrix Factorization with implicit feedback
- **Content-Based**: Product similarity using cosine similarity on attributes
- **Hybrid**: Weighted combination (60% collaborative + 40% content-based)

### Evaluation Metrics

- Offline: MAE, RMSE, Coverage, Precision@K, Recall@K
- Online: CTR, conversion rate, revenue impact via A/B testing

## Common Issues

This is a documentation repository with no runnable code, so typical development commands (build, test, run) don't apply. The skills are consumed by GitHub Copilot when generating code in actual implementation repositories.

If working on the skills themselves:
- Validate YAML frontmatter in SKILL.md files
- Ensure code examples follow the documented conventions
- Test that instructions are clear and actionable
