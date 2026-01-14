# Copilot Agent Skills - E-commerce Recommendation System

Agent skills for GitHub Copilot to support development of recommendation systems for e-commerce.

## 🚀 Quick Start

### Enable Agent Skills in VS Code
1. Open VS Code Settings (`Cmd + ,`)
2. Search for `chat.useAgentSkills`
3. Enable this setting

### Project Structure
```
.github/
├── copilot-instructions.md              # Project-wide instructions
└── skills/
    ├── airflow-dag/SKILL.md             # DAG development patterns
    ├── bigquery/SKILL.md                # Query & feature engineering
    ├── composer/SKILL.md                # GCP Composer deployment
    ├── jenkins-cicd/SKILL.md            # CI/CD pipelines
    ├── pytest-testing/SKILL.md          # Testing data pipelines
    └── recommendation-ml/SKILL.md       # ML model development
```

## 📚 Available Skills

| Skill | Description | Use When |
|-------|-------------|----------|
| **Airflow DAG** | Templates and patterns for DAG development | Create/edit DAGs, write tasks |
| **BigQuery** | Query optimization, feature engineering | Write SQL, create features for ML |
| **Composer** | GCP Composer deployment | Setup/configure Composer |
| **Jenkins CI/CD** | Pipeline templates | Create Jenkinsfile, CI/CD |
| **Pytest Testing** | Testing patterns | Write tests for DAGs/queries |
| **Recommendation ML** | ML model development | Build recommendation models |

## 💬 Example Prompts

### Airflow DAG
```
Create a DAG to extract user events from BigQuery, transform features,
and load into recommendation table. Run daily at 2 AM.
```

### BigQuery Feature Engineering
```
Write a query to create user features for recommendation system,
including recency, frequency, monetary metrics.
```

### Recommendation Model
```
Create a Matrix Factorization model with BigQuery ML
to recommend products to users.
```

### Testing
```
Write unit tests for recommendation DAG,
including test structure and task logic.
```

### CI/CD Pipeline
```
Create a Jenkinsfile to validate DAGs, run tests,
and deploy to Composer staging/production.
```

## 🛠 Tech Stack

- **Orchestration**: Apache Airflow 2.x on GCP Composer 2
- **Data Warehouse**: Google BigQuery
- **ML Platform**: BigQuery ML, Vertex AI
- **CI/CD**: Jenkins
- **Testing**: Pytest
- **Language**: Python 3.10+

## 📖 Documentation

- [Airflow Best Practices](https://cloud.google.com/composer/docs/how-to/using/writing-dags)
- [BigQuery ML](https://cloud.google.com/bigquery/docs/bqml-introduction)
- [Copilot Agent Skills](https://code.visualstudio.com/docs/copilot/copilot-agent-skills)

## 🔧 Configuration

### VS Code Settings
```json
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  "chat.useAgentSkills": true
}
```

### Recommended Extensions
- GitHub Copilot
- GitHub Copilot Chat
- Python
- Google Cloud Code
