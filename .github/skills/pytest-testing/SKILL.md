---
name: Pytest Testing
description: Support for writing tests for Airflow DAGs, BigQuery queries and data pipelines
---

# Pytest Testing Skill

This skill guides Copilot to write effective tests for data pipelines.

## When to Use
- Write unit tests for DAG tasks
- Test BigQuery queries with mocks
- Create integration tests
- Data validation tests

## Project Setup

### pytest.ini
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short --strict-markers
markers =
    unit: Unit tests
    integration: Integration tests (require GCP credentials)
    slow: Slow tests
filterwarnings =
    ignore::DeprecationWarning
```

### conftest.py
```python
# tests/conftest.py
import pytest
from datetime import datetime
from unittest.mock import MagicMock, patch
from airflow.models import DAG, Connection
from airflow.utils.state import State

@pytest.fixture
def mock_dag():
    """Create a mock DAG for testing."""
    return DAG(
        dag_id="test_dag",
        start_date=datetime(2024, 1, 1),
        schedule=None,
        catchup=False,
    )

@pytest.fixture
def mock_task_instance(mock_dag):
    """Create a mock TaskInstance."""
    from airflow.models import TaskInstance
    from airflow.operators.empty import EmptyOperator
    
    task = EmptyOperator(task_id="test_task", dag=mock_dag)
    ti = TaskInstance(task=task, run_id="test_run")
    ti.state = State.RUNNING
    return ti

@pytest.fixture
def mock_context(mock_dag, mock_task_instance):
    """Create a mock Airflow context."""
    return {
        "dag": mock_dag,
        "task_instance": mock_task_instance,
        "ds": "2024-01-01",
        "ds_nodash": "20240101",
        "execution_date": datetime(2024, 1, 1),
        "params": {},
    }

@pytest.fixture
def mock_bigquery_client():
    """Mock BigQuery client."""
    with patch("google.cloud.bigquery.Client") as mock:
        client = MagicMock()
        mock.return_value = client
        yield client

@pytest.fixture
def mock_gcs_client():
    """Mock GCS client."""
    with patch("google.cloud.storage.Client") as mock:
        client = MagicMock()
        mock.return_value = client
        yield client
```

## DAG Testing

### Test DAG Structure
```python
# tests/unit/dags/test_recommendation_dag.py
import pytest
from airflow.models import DagBag

class TestRecommendationDAG:
    """Tests for recommendation DAG."""
    
    @pytest.fixture
    def dag_bag(self):
        return DagBag(dag_folder="src/dags", include_examples=False)
    
    def test_dag_loaded(self, dag_bag):
        """Test DAG loads without errors."""
        assert "recommendation_train_daily" in dag_bag.dags
        assert len(dag_bag.import_errors) == 0
    
    def test_dag_tags(self, dag_bag):
        """Test DAG has required tags."""
        dag = dag_bag.get_dag("recommendation_train_daily")
        assert "recommendation" in dag.tags
        assert "daily" in dag.tags
    
    def test_dag_schedule(self, dag_bag):
        """Test DAG schedule is correct."""
        dag = dag_bag.get_dag("recommendation_train_daily")
        assert dag.schedule_interval == "0 2 * * *"
    
    def test_dag_catchup_disabled(self, dag_bag):
        """Test catchup is disabled."""
        dag = dag_bag.get_dag("recommendation_train_daily")
        assert dag.catchup is False
    
    def test_dag_default_args(self, dag_bag):
        """Test default args are properly set."""
        dag = dag_bag.get_dag("recommendation_train_daily")
        assert dag.default_args["retries"] >= 1
        assert dag.default_args["owner"] != "airflow"
    
    def test_task_dependencies(self, dag_bag):
        """Test task dependencies are correct."""
        dag = dag_bag.get_dag("recommendation_train_daily")
        
        extract_task = dag.get_task("extract_user_events")
        transform_task = dag.get_task("transform_features")
        load_task = dag.get_task("load_to_bigquery")
        
        # Check upstream dependencies
        assert extract_task in transform_task.upstream_list
        assert transform_task in load_task.upstream_list
```

### Test DAG Tasks
```python
# tests/unit/dags/test_dag_tasks.py
import pytest
from unittest.mock import patch, MagicMock
from src.dags.recommendation.tasks import (
    extract_user_events,
    transform_features,
    calculate_recommendations,
)

class TestRecommendationTasks:
    """Tests for recommendation DAG tasks."""
    
    def test_extract_user_events_success(self, mock_context, mock_bigquery_client):
        """Test extract_user_events returns expected data."""
        # Setup mock
        mock_bigquery_client.query.return_value.to_dataframe.return_value = MagicMock(
            empty=False,
            shape=(1000, 5)
        )
        
        # Execute
        result = extract_user_events(**mock_context)
        
        # Assert
        assert result is not None
        mock_bigquery_client.query.assert_called_once()
    
    def test_extract_user_events_empty_result(self, mock_context, mock_bigquery_client):
        """Test extract_user_events handles empty result."""
        mock_bigquery_client.query.return_value.to_dataframe.return_value = MagicMock(
            empty=True
        )
        
        with pytest.raises(ValueError, match="No data found"):
            extract_user_events(**mock_context)
    
    def test_transform_features(self, mock_context):
        """Test feature transformation logic."""
        input_data = [
            {"user_id": "u1", "product_id": "p1", "event_type": "view"},
            {"user_id": "u1", "product_id": "p1", "event_type": "purchase"},
        ]
        
        result = transform_features(input_data, **mock_context)
        
        assert len(result) == 1  # Aggregated by user-product
        assert result[0]["interaction_score"] == 6  # 1 view + 5 purchase
```

## BigQuery Query Testing

```python
# tests/unit/sql/test_feature_queries.py
import pytest
from pathlib import Path

class TestFeatureQueries:
    """Tests for BigQuery feature engineering queries."""
    
    @pytest.fixture
    def sql_content(self):
        sql_file = Path("src/sql/features/user_features.sql")
        return sql_file.read_text()
    
    def test_query_has_partition_filter(self, sql_content):
        """Ensure query includes partition filter for cost optimization."""
        assert "WHERE" in sql_content.upper()
        assert any(
            pattern in sql_content.lower()
            for pattern in ["date(", "timestamp_", "_partitiontime"]
        )
    
    def test_query_no_select_star(self, sql_content):
        """Ensure query doesn't use SELECT *."""
        lines = sql_content.split("\n")
        for line in lines:
            if not line.strip().startswith("--"):  # Skip comments
                assert "SELECT *" not in line.upper(), \
                    "Query should specify columns instead of SELECT *"
    
    def test_query_uses_safe_divide(self, sql_content):
        """Ensure division uses SAFE_DIVIDE."""
        if "/" in sql_content:
            # Allow / in comments or SAFE_DIVIDE
            assert "SAFE_DIVIDE" in sql_content or \
                   all("--" in line for line in sql_content.split("\n") if "/" in line)
```

## Data Validation Tests

```python
# tests/integration/test_data_quality.py
import pytest
from google.cloud import bigquery

@pytest.mark.integration
class TestDataQuality:
    """Integration tests for data quality validation."""
    
    @pytest.fixture
    def bq_client(self):
        return bigquery.Client()
    
    def test_user_features_not_null(self, bq_client):
        """Test critical columns are not null."""
        query = """
        SELECT COUNT(*) as null_count
        FROM `project.recommendation.user_features`
        WHERE user_id IS NULL OR total_events IS NULL
        """
        result = list(bq_client.query(query).result())[0]
        assert result.null_count == 0, "Found null values in critical columns"
    
    def test_interaction_scores_valid(self, bq_client):
        """Test interaction scores are within valid range."""
        query = """
        SELECT 
            MIN(interaction_score) as min_score,
            MAX(interaction_score) as max_score
        FROM `project.recommendation.user_product_interactions`
        """
        result = list(bq_client.query(query).result())[0]
        assert result.min_score > 0, "Interaction scores must be positive"
        assert result.max_score < 1000, "Abnormally high interaction score"
    
    def test_no_duplicate_user_products(self, bq_client):
        """Test no duplicate user-product pairs."""
        query = """
        SELECT user_id, product_id, COUNT(*) as cnt
        FROM `project.recommendation.user_product_interactions`
        GROUP BY user_id, product_id
        HAVING cnt > 1
        """
        result = list(bq_client.query(query).result())
        assert len(result) == 0, f"Found {len(result)} duplicate user-product pairs"
```

## Advanced Fixtures & Mocking

### Enhanced conftest.py
```python
# tests/conftest.py
import pytest
from datetime import datetime
from unittest.mock import MagicMock, patch
from freezegun import freeze_time
from faker import Faker
import pandas as pd

@pytest.fixture
def fake():
    """Faker instance for generating test data."""
    return Faker()

@pytest.fixture
def freeze_at_date():
    """Freeze time at specific date for testing."""
    with freeze_time("2024-01-15 10:00:00"):
        yield

@pytest.fixture
def mock_bigquery_dataset():
    """Mock BigQuery dataset with realistic structure."""
    dataset = MagicMock()
    dataset.dataset_id = "test_dataset"
    dataset.project = "test_project"

    # Mock tables with metadata
    tables = {}
    for table_name in ["user_events", "user_features", "product_features"]:
        table = MagicMock()
        table.table_id = table_name
        table.num_rows = 1000
        table.created = datetime(2024, 1, 1)
        tables[table_name] = table

    dataset.tables = tables
    return dataset

@pytest.fixture
def sample_user_events_df(fake) -> pd.DataFrame:
    """Generate realistic sample user events DataFrame."""
    n_rows = 100
    return pd.DataFrame({
        'user_id': [fake.uuid4() for _ in range(n_rows)],
        'product_id': [f'prod_{fake.random_int(1, 100)}' for _ in range(n_rows)],
        'event_type': [fake.random_element(['view', 'add_to_cart', 'purchase'])
                       for _ in range(n_rows)],
        'event_timestamp': [fake.date_time_between(start_date='-30d', end_date='now')
                           for _ in range(n_rows)],
        'session_id': [fake.uuid4() for _ in range(n_rows)],
        'purchase_amount': [fake.random_int(10, 1000)
                           if fake.boolean(chance_of_getting_true=20) else None
                           for _ in range(n_rows)],
    })

@pytest.fixture
def airflow_test_env(monkeypatch):
    """Set up Airflow test environment variables."""
    test_vars = {
        'AIRFLOW__CORE__DAGS_FOLDER': '/tmp/test_dags',
        'AIRFLOW__CORE__LOAD_EXAMPLES': 'False',
        'AIRFLOW__CORE__UNIT_TEST_MODE': 'True',
        'AIRFLOW_VAR_GCP_PROJECT': 'test-project',
        'AIRFLOW_VAR_GCP_REGION': 'us-central1',
    }
    for key, value in test_vars.items():
        monkeypatch.setenv(key, value)
```

### Parametrized Fixtures
```python
@pytest.fixture(params=['view', 'add_to_cart', 'purchase'])
def event_type(request):
    """Parametrized event type fixture."""
    return request.param

@pytest.fixture(params=[10, 100, 1000])
def batch_size(request):
    """Parametrized batch size for testing scalability."""
    return request.param

def test_process_events_all_types(event_type, sample_user_events_df):
    """Test processing works for all event types."""
    df = sample_user_events_df[sample_user_events_df['event_type'] == event_type]
    result = process_events(df)
    assert result is not None
```

## Property-Based Testing

### Using Hypothesis
```python
# tests/unit/test_feature_engineering.py
import pytest
from hypothesis import given, strategies as st, assume
from src.feature_engineering import calculate_interaction_score, aggregate_user_features

@given(
    views=st.integers(min_value=0, max_value=1000),
    cart_adds=st.integers(min_value=0, max_value=100),
    purchases=st.integers(min_value=0, max_value=10)
)
def test_interaction_score_properties(views, cart_adds, purchases):
    """Test interaction score calculation properties."""
    score = calculate_interaction_score(views, cart_adds, purchases)

    # Score should always be non-negative
    assert score >= 0

    # Purchases should contribute more than cart adds
    if purchases > 0:
        score_with_purchase = calculate_interaction_score(0, 0, purchases)
        score_with_cart = calculate_interaction_score(0, cart_adds, 0)
        assert score_with_purchase > score_with_cart or cart_adds == 0

    # Score should be monotonic
    score_double_views = calculate_interaction_score(views * 2, cart_adds, purchases)
    assert score_double_views >= score

@given(
    user_events=st.lists(
        st.fixed_dictionaries({
            'user_id': st.text(min_size=1, max_size=36),
            'event_type': st.sampled_from(['view', 'add_to_cart', 'purchase']),
            'product_id': st.text(min_size=1, max_size=10),
            'timestamp': st.datetimes(),
        }),
        min_size=1,
        max_size=100
    )
)
def test_user_aggregation_properties(user_events):
    """Test user feature aggregation properties."""
    result = aggregate_user_features(user_events)

    # Each user should appear once
    user_ids = [r['user_id'] for r in result]
    assert len(user_ids) == len(set(user_ids))

    # Event counts should match
    for user_result in result:
        user_id = user_result['user_id']
        expected_events = sum(1 for e in user_events if e['user_id'] == user_id)
        assert user_result['total_events'] == expected_events
```

## Performance & Load Testing

### Benchmarking with pytest-benchmark
```python
# tests/performance/test_query_performance.py
import pytest
from google.cloud import bigquery

@pytest.mark.performance
class TestQueryPerformance:
    """Performance tests for BigQuery queries."""

    @pytest.fixture
    def bq_client(self):
        return bigquery.Client()

    def test_user_features_query_performance(self, bq_client, benchmark):
        """Benchmark user features query."""

        def run_query():
            query = """
            SELECT user_id, COUNT(*) as events
            FROM `project.raw.user_events`
            WHERE DATE(event_timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
            GROUP BY user_id
            LIMIT 1000
            """
            return list(bq_client.query(query).result())

        # Benchmark the query
        result = benchmark(run_query)

        # Assert performance requirements
        assert benchmark.stats.stats.mean < 5.0  # Should complete in <5 seconds
        assert len(result) > 0

    @pytest.mark.slow
    def test_query_cost_regression(self, bq_client):
        """Test query doesn't exceed cost budget."""
        query = """
        SELECT * FROM `project.recommendation.user_features`
        WHERE DATE(created_at) = CURRENT_DATE()
        """

        # Dry run to estimate cost
        job_config = bigquery.QueryJobConfig(dry_run=True)
        query_job = bq_client.query(query, job_config=job_config)

        bytes_processed = query_job.total_bytes_processed
        estimated_cost_usd = (bytes_processed / 1e12) * 5  # $5 per TB

        assert estimated_cost_usd < 0.10, f"Query too expensive: ${estimated_cost_usd:.4f}"
```

## Contract Testing

### Schema Validation with Pydantic
```python
# tests/integration/test_data_contracts.py
import pytest
from google.cloud import bigquery
from pydantic import BaseModel, Field, validator
from datetime import datetime

class UserFeatureSchema(BaseModel):
    """Contract for user_features table."""
    user_id: str = Field(..., min_length=1, max_length=36)
    total_events: int = Field(..., ge=0)
    total_spent: float = Field(..., ge=0)
    days_since_last_visit: int = Field(..., ge=0, le=365)
    created_at: datetime

    @validator('total_spent')
    def spent_reasonable(cls, v):
        if v > 1000000:
            raise ValueError('Unreasonably high spend')
        return v

@pytest.mark.integration
class TestDataContracts:
    """Test data conforms to expected schemas."""

    def test_user_features_schema(self, bq_client):
        """Validate user_features table schema."""
        query = """
        SELECT * FROM `project.recommendation.user_features`
        LIMIT 100
        """
        results = bq_client.query(query).result()

        errors = []
        for row in results:
            try:
                UserFeatureSchema(**dict(row))
            except Exception as e:
                errors.append(f"Row validation failed: {e}")

        assert len(errors) == 0, f"Schema violations: {errors}"

    def test_schema_backward_compatibility(self, bq_client):
        """Test new schema is backward compatible."""
        table = bq_client.get_table("project.recommendation.user_features")
        current_fields = {field.name for field in table.schema}

        # Required fields that must exist
        required_fields = {'user_id', 'total_events', 'created_at'}

        assert required_fields.issubset(current_fields), \
            f"Missing required fields: {required_fields - current_fields}"
```

## Edge Case Testing

### Comprehensive Edge Cases
```python
# tests/unit/test_edge_cases.py
import pytest
import numpy as np
import pandas as pd
from src.feature_engineering import calculate_rfm_features

class TestEdgeCases:
    """Test edge cases and boundary conditions."""

    def test_empty_dataframe(self):
        """Test handling of empty input."""
        df = pd.DataFrame(columns=['user_id', 'event_type', 'timestamp'])
        result = calculate_rfm_features(df)
        assert len(result) == 0

    def test_single_event(self):
        """Test with single event."""
        df = pd.DataFrame([{
            'user_id': 'user1',
            'event_type': 'purchase',
            'timestamp': pd.Timestamp('2024-01-01'),
            'amount': 100.0
        }])
        result = calculate_rfm_features(df)
        assert len(result) == 1
        assert result.iloc[0]['recency_days'] >= 0

    def test_null_values(self):
        """Test handling of null values."""
        df = pd.DataFrame([
            {'user_id': 'user1', 'amount': None},
            {'user_id': 'user2', 'amount': 100.0},
        ])
        result = calculate_rfm_features(df)
        # Should handle nulls gracefully
        assert not result['total_spent'].isna().any()

    def test_extreme_values(self):
        """Test with extreme input values."""
        df = pd.DataFrame([
            {'user_id': 'user1', 'amount': 1e10},  # Very large
            {'user_id': 'user2', 'amount': 0.01},  # Very small
        ])
        result = calculate_rfm_features(df)
        assert all(np.isfinite(result['total_spent']))

    @pytest.mark.parametrize("event_type,expected_score", [
        ('view', 1),
        ('add_to_cart', 3),
        ('purchase', 5),
        ('unknown', 0),
    ])
    def test_event_type_scores(self, event_type, expected_score):
        """Test scoring for different event types."""
        score = calculate_interaction_score(event_type)
        assert score == expected_score
```

## Test Data Management

### Test Data Factories
```python
# tests/factories.py
import factory
from factory import fuzzy
from datetime import datetime

class UserEventFactory(factory.Factory):
    """Factory for generating test user events."""

    class Meta:
        model = dict

    user_id = factory.Faker('uuid4')
    product_id = factory.LazyAttribute(lambda o: f"prod_{fuzzy.FuzzyInteger(1, 1000).fuzz()}")
    event_type = fuzzy.FuzzyChoice(['view', 'add_to_cart', 'purchase'])
    event_timestamp = factory.Faker('date_time_between', start_date='-30d', end_date='now')
    session_id = factory.Faker('uuid4')

    @factory.lazy_attribute
    def purchase_amount(self):
        if self.event_type == 'purchase':
            return fuzzy.FuzzyFloat(10.0, 1000.0).fuzz()
        return None

class UserFeatureFactory(factory.Factory):
    """Factory for user features."""

    class Meta:
        model = dict

    user_id = factory.Faker('uuid4')
    total_events = fuzzy.FuzzyInteger(1, 1000)
    total_spent = fuzzy.FuzzyFloat(0, 10000)
    days_since_last_visit = fuzzy.FuzzyInteger(0, 90)
    preferred_categories = factory.LazyFunction(
        lambda: [fuzzy.FuzzyChoice(['tops', 'bottoms', 'outerwear']).fuzz()
                 for _ in range(3)]
    )

# Usage in tests
def test_with_factory_data():
    """Test using factory-generated data."""
    events = UserEventFactory.create_batch(100)
    user_features = UserFeatureFactory.create_batch(50)

    # Process data
    result = process_user_data(events, user_features)
    assert len(result) > 0
```

### Synthetic Data Generation
```python
# tests/utils/synthetic_data.py
import pandas as pd
from faker import Faker
from typing import List
import random

def generate_realistic_user_events(n_users: int = 100, n_events: int = 1000) -> pd.DataFrame:
    """Generate realistic user event data for testing."""
    fake = Faker()

    # Create user pool
    users = [fake.uuid4() for _ in range(n_users)]
    products = [f"prod_{i}" for i in range(1, 201)]

    events = []
    for _ in range(n_events):
        user_id = random.choice(users)
        product_id = random.choice(products)
        event_type = random.choices(
            ['view', 'add_to_cart', 'purchase'],
            weights=[70, 20, 10]  # Realistic conversion funnel
        )[0]

        events.append({
            'user_id': user_id,
            'product_id': product_id,
            'event_type': event_type,
            'event_timestamp': fake.date_time_between(start_date='-30d'),
            'session_id': fake.uuid4(),
            'purchase_amount': fake.random_int(10, 1000) if event_type == 'purchase' else None,
        })

    return pd.DataFrame(events)
```

## Best Practices
- Separate unit and integration tests with markers
- Use fixtures for common mocks (BigQuery, GCS clients)
- Test DAG structure, not just task logic
- Validate SQL queries for common issues (SELECT *, partitioning)
- Include data quality tests in CI pipeline
- Mock external services in unit tests
- Use property-based testing to find edge cases automatically
- Benchmark performance-critical code with pytest-benchmark
- Validate data contracts with Pydantic schemas
- Test with realistic synthetic data using factories
- Freeze time for deterministic date-based tests
- Parametrize tests for multiple scenarios
- Test edge cases: empty data, nulls, extreme values
- Use hypothesis for property-based testing
- Implement contract tests for schema validation
- Generate realistic test data with Faker and factories
- Track test coverage and aim for >80% for critical paths
