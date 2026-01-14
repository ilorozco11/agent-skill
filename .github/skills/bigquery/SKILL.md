---
name: BigQuery Development
description: Support for writing BigQuery queries, feature engineering and ML models for recommendation systems
---

# BigQuery Development Skill

This skill guides Copilot to write optimized BigQuery queries for recommendation systems.

## When to Use
- Write SQL queries for BigQuery
- Feature engineering for ML models
- Create BigQuery ML models
- Optimize query performance

## Query Optimization

### 1. Avoid SELECT * - Specify Columns
```sql
-- ❌ Bad
SELECT * FROM `project.dataset.user_events`

-- ✅ Good
SELECT 
    user_id,
    event_type,
    product_id,
    event_timestamp
FROM `project.dataset.user_events`
WHERE DATE(event_timestamp) = '2024-01-01'
```

### 2. Partitioning and Clustering
```sql
-- Create partitioned and clustered table
CREATE OR REPLACE TABLE `project.recommendation.user_features`
PARTITION BY DATE(created_at)
CLUSTER BY user_id, product_category
AS
SELECT
    user_id,
    product_category,
    COUNT(*) as view_count,
    SUM(purchase_amount) as total_spent,
    CURRENT_TIMESTAMP() as created_at
FROM `project.raw.user_events`
GROUP BY user_id, product_category
```

### 3. Use Materialized Views
```sql
CREATE MATERIALIZED VIEW `project.recommendation.daily_product_stats`
PARTITION BY DATE(stat_date)
CLUSTER BY product_id
AS
SELECT
    DATE(event_timestamp) as stat_date,
    product_id,
    COUNT(DISTINCT user_id) as unique_viewers,
    SUM(IF(event_type = 'purchase', 1, 0)) as purchases
FROM `project.raw.user_events`
GROUP BY stat_date, product_id
```

## Feature Engineering

### User Features
```sql
-- User behavior features
CREATE OR REPLACE TABLE `project.recommendation.user_features` AS
SELECT
    user_id,
    -- Recency
    DATE_DIFF(CURRENT_DATE(), MAX(DATE(event_timestamp)), DAY) as days_since_last_visit,
    -- Frequency
    COUNT(DISTINCT DATE(event_timestamp)) as visit_days,
    COUNT(*) as total_events,
    -- Monetary
    SUM(IF(event_type = 'purchase', purchase_amount, 0)) as total_spent,
    COUNTIF(event_type = 'purchase') as purchase_count,
    -- Category preferences
    ARRAY_AGG(DISTINCT product_category LIMIT 10) as preferred_categories,
    -- Session metrics
    AVG(session_duration) as avg_session_duration
FROM `project.raw.user_events`
WHERE event_timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY user_id
```

### Product Features
```sql
-- Product popularity features
CREATE OR REPLACE TABLE `project.recommendation.product_features` AS
SELECT
    product_id,
    product_name,
    category,
    price,
    -- Popularity metrics
    COUNT(DISTINCT user_id) as unique_viewers,
    COUNTIF(event_type = 'add_to_cart') as cart_adds,
    COUNTIF(event_type = 'purchase') as purchases,
    -- Conversion rate
    SAFE_DIVIDE(
        COUNTIF(event_type = 'purchase'),
        COUNTIF(event_type = 'view')
    ) as view_to_purchase_rate,
    -- Rating
    AVG(rating) as avg_rating,
    COUNT(rating) as review_count
FROM `project.raw.user_events` e
JOIN `project.catalog.products` p USING(product_id)
GROUP BY product_id, product_name, category, price
```

### Interaction Features
```sql
-- User-Product interaction matrix
CREATE OR REPLACE TABLE `project.recommendation.user_product_interactions` AS
SELECT
    user_id,
    product_id,
    -- Implicit feedback score
    COUNTIF(event_type = 'view') * 1 +
    COUNTIF(event_type = 'add_to_cart') * 3 +
    COUNTIF(event_type = 'purchase') * 5 as interaction_score,
    MAX(event_timestamp) as last_interaction
FROM `project.raw.user_events`
WHERE event_timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY user_id, product_id
HAVING interaction_score > 0
```

## BigQuery ML - Matrix Factorization

```sql
-- Create recommendation model
CREATE OR REPLACE MODEL `project.recommendation.product_recommender`
OPTIONS(
    model_type = 'MATRIX_FACTORIZATION',
    feedback_type = 'IMPLICIT',
    user_col = 'user_id',
    item_col = 'product_id',
    rating_col = 'interaction_score',
    num_factors = 50,
    l2_reg = 0.1,
    max_iterations = 20
) AS
SELECT
    user_id,
    product_id,
    interaction_score
FROM `project.recommendation.user_product_interactions`
```

### Generate Recommendations
```sql
-- Get top 10 recommendations per user
SELECT
    user_id,
    ARRAY_AGG(
        STRUCT(product_id, predicted_rating)
        ORDER BY predicted_rating DESC
        LIMIT 10
    ) as recommendations
FROM ML.RECOMMEND(
    MODEL `project.recommendation.product_recommender`,
    (SELECT DISTINCT user_id FROM `project.recommendation.active_users`)
)
GROUP BY user_id
```

## Advanced Query Optimization

### Query Performance Analysis
```sql
-- Analyze query performance from INFORMATION_SCHEMA
SELECT
  job_id,
  user_email,
  query,
  total_bytes_processed,
  total_slot_ms,
  total_bytes_billed,
  TIMESTAMP_DIFF(end_time, start_time, MILLISECOND) as duration_ms,
  (total_bytes_billed / POW(10, 12)) * 5 as estimated_cost_usd,
  cache_hit
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
ORDER BY total_bytes_processed DESC
LIMIT 100;
```

### Skew Detection and Optimization
```sql
-- Detect hot keys that cause data skew
SELECT
  user_id,
  COUNT(*) as event_count
FROM `project.raw.user_events`
GROUP BY user_id
HAVING event_count > 10000
ORDER BY event_count DESC;

-- Use approximate aggregation for large datasets
SELECT
  user_id,
  APPROX_COUNT_DISTINCT(session_id) as approx_sessions,
  APPROX_QUANTILES(purchase_amount, 100)[OFFSET(50)] as median_purchase,
  APPROX_TOP_COUNT(product_category, 5) as top_categories
FROM `project.raw.user_events`
WHERE DATE(event_timestamp) >= CURRENT_DATE() - 30
GROUP BY user_id;
```

### Window Function Optimization
```sql
-- Optimized window functions with frame clauses
SELECT
  user_id,
  event_date,
  daily_events,
  -- Use specific frame instead of default (more efficient)
  AVG(daily_events) OVER (
    PARTITION BY user_id
    ORDER BY event_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as events_7day_avg,
  -- Use RANGE for time-based windows
  SUM(daily_revenue) OVER (
    PARTITION BY user_id
    ORDER BY UNIX_DATE(event_date)
    RANGE BETWEEN 29 PRECEDING AND CURRENT ROW
  ) as revenue_30day_sum
FROM user_daily_metrics;
```

## Cost Management & Monitoring

### Query Cost Attribution
```sql
-- Track costs by team/project
WITH query_costs AS (
  SELECT
    user_email,
    REGEXP_EXTRACT(query, r'`([^.]+)\.') as project,
    REGEXP_EXTRACT(query, r'`[^.]+\.([^.]+)\.') as dataset,
    COUNT(*) as query_count,
    SUM(total_bytes_billed) / POW(10, 12) as tb_billed,
    SUM(total_bytes_billed) / POW(10, 12) * 5 as cost_usd
  FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
  WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND job_type = 'QUERY'
    AND state = 'DONE'
  GROUP BY user_email, project, dataset
)
SELECT
  user_email,
  dataset,
  query_count,
  ROUND(cost_usd, 2) as cost_usd,
  ROUND(cost_usd / SUM(cost_usd) OVER () * 100, 2) as cost_percentage
FROM query_costs
ORDER BY cost_usd DESC
LIMIT 50;
```

### Budget Alerts and Quotas
```sql
-- Monitor daily quota usage
SELECT
  DATE(creation_time) as query_date,
  SUM(total_bytes_billed) / POW(10, 9) as total_gb_billed,
  -- Assuming 100 GB daily quota
  (SUM(total_bytes_billed) / POW(10, 9)) / 100 * 100 as quota_usage_percent,
  CASE
    WHEN (SUM(total_bytes_billed) / POW(10, 9)) > 90 THEN 'CRITICAL'
    WHEN (SUM(total_bytes_billed) / POW(10, 9)) > 70 THEN 'WARNING'
    ELSE 'OK'
  END as alert_level
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE DATE(creation_time) = CURRENT_DATE()
  AND job_type = 'QUERY'
GROUP BY query_date;

-- Set custom quota (via bq CLI)
-- bq update --daily_bytes_billed_limit=107374182400 project:dataset
```

## Advanced Time-Series Features

### Lag and Rolling Windows
```sql
CREATE OR REPLACE TABLE `project.features.user_time_series` AS
SELECT
  user_id,
  event_date,
  daily_events,
  daily_revenue,
  -- Lag features (previous values)
  LAG(daily_events, 1) OVER w_user as events_yesterday,
  LAG(daily_events, 7) OVER w_user as events_last_week,
  -- Rolling windows
  AVG(daily_events) OVER w7 as events_7day_avg,
  STDDEV(daily_events) OVER w7 as events_7day_stddev,
  SUM(daily_revenue) OVER w30 as revenue_30day_sum,
  -- Exponential moving average
  AVG(daily_events) OVER (
    PARTITION BY user_id
    ORDER BY event_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as ema_7day,
  -- Trend detection with linear regression
  REGR_SLOPE(daily_events, DATE_DIFF(event_date, DATE('2024-01-01'), DAY))
    OVER w30 as trend_slope,
  REGR_INTERCEPT(daily_events, DATE_DIFF(event_date, DATE('2024-01-01'), DAY))
    OVER w30 as trend_intercept
FROM user_daily_aggregates
WINDOW
  w7 AS (PARTITION BY user_id ORDER BY event_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW),
  w30 AS (PARTITION BY user_id ORDER BY event_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW),
  w_user AS (PARTITION BY user_id ORDER BY event_date);
```

### Seasonality Features
```sql
SELECT
  user_id,
  event_date,
  -- Cyclical encoding for time features
  SIN(2 * ACOS(-1) * EXTRACT(DAYOFWEEK FROM event_date) / 7) as day_of_week_sin,
  COS(2 * ACOS(-1) * EXTRACT(DAYOFWEEK FROM event_date) / 7) as day_of_week_cos,
  SIN(2 * ACOS(-1) * EXTRACT(MONTH FROM event_date) / 12) as month_sin,
  COS(2 * ACOS(-1) * EXTRACT(MONTH FROM event_date) / 12) as month_cos,
  -- Seasonal indicators
  CASE EXTRACT(DAYOFWEEK FROM event_date)
    WHEN 1 THEN 1 WHEN 7 THEN 1 ELSE 0
  END as is_weekend,
  CASE
    WHEN EXTRACT(MONTH FROM event_date) IN (11, 12) THEN 1 ELSE 0
  END as is_holiday_season
FROM user_events;
```

## BigQuery ML Advanced Features

### Model Explainability
```sql
-- Feature importance and explanations
SELECT
  user_id,
  product_id,
  predicted_rating,
  -- Extract top contributing features
  ARRAY(
    SELECT AS STRUCT
      feature,
      importance,
      CASE feature
        WHEN 'user_total_purchases' THEN 'Frequent buyer'
        WHEN 'product_popularity' THEN 'Trending item'
        WHEN 'category_affinity' THEN 'Matches preferences'
        ELSE 'Related to past interactions'
      END as explanation
    FROM UNNEST(ml_explain_features)
    ORDER BY importance DESC
    LIMIT 3
  ) as top_reasons
FROM ML.EXPLAIN_PREDICT(
  MODEL `project.recommendation.product_recommender`,
  (SELECT DISTINCT user_id FROM `project.recommendation.active_users` LIMIT 100),
  STRUCT(5 AS top_k_features)
);
```

### Incremental Model Training
```sql
-- Create initial model
CREATE OR REPLACE MODEL `project.recommendation.product_recommender_v1`
OPTIONS(
  model_type = 'MATRIX_FACTORIZATION',
  feedback_type = 'IMPLICIT',
  user_col = 'user_id',
  item_col = 'product_id',
  rating_col = 'interaction_score'
) AS
SELECT user_id, product_id, interaction_score
FROM `project.recommendation.user_product_interactions`
WHERE DATE(last_interaction) <= '2024-01-31';

-- Warm start from existing model with new data
CREATE OR REPLACE MODEL `project.recommendation.product_recommender_v2`
OPTIONS(
  model_type = 'MATRIX_FACTORIZATION',
  feedback_type = 'IMPLICIT',
  user_col = 'user_id',
  item_col = 'product_id',
  rating_col = 'interaction_score',
  warm_start = TRUE,
  num_factors = 50
) AS
SELECT user_id, product_id, interaction_score
FROM `project.recommendation.user_product_interactions`
WHERE DATE(last_interaction) > '2024-01-31';
```

### Model Evaluation Metrics
```sql
-- Calculate comprehensive evaluation metrics
WITH predictions AS (
  SELECT
    actual.user_id,
    actual.product_id,
    actual.interaction_score as actual_score,
    pred.predicted_rating as predicted_score
  FROM `project.recommendation.test_interactions` actual
  JOIN ML.PREDICT(
    MODEL `project.recommendation.product_recommender`,
    TABLE `project.recommendation.test_interactions`
  ) pred
  USING (user_id, product_id)
)
SELECT
  -- Regression metrics
  AVG(ABS(actual_score - predicted_score)) as mae,
  SQRT(AVG(POW(actual_score - predicted_score, 2))) as rmse,
  CORR(actual_score, predicted_score) as correlation,
  -- Coverage
  COUNT(DISTINCT product_id) / (
    SELECT COUNT(DISTINCT product_id)
    FROM `project.catalog.products`
  ) as catalog_coverage
FROM predictions;
```

## Data Quality & Validation

### Comprehensive Data Quality Checks
```sql
CREATE OR REPLACE TABLE `project.monitoring.data_quality_checks` AS
WITH validation_results AS (
  -- Null checks
  SELECT
    'user_features' as table_name,
    'user_id' as column_name,
    'null_check' as check_type,
    COUNTIF(user_id IS NULL) as failures,
    COUNT(*) as total_rows
  FROM `project.recommendation.user_features`

  UNION ALL

  -- Range checks
  SELECT
    'user_features',
    'total_spent',
    'range_check',
    COUNTIF(total_spent < 0 OR total_spent > 1000000),
    COUNT(*)
  FROM `project.recommendation.user_features`

  UNION ALL

  -- Uniqueness checks
  SELECT
    'user_features',
    'user_id',
    'uniqueness_check',
    COUNT(*) - COUNT(DISTINCT user_id),
    COUNT(*)
  FROM `project.recommendation.user_features`

  UNION ALL

  -- Freshness checks
  SELECT
    'user_features',
    'created_at',
    'freshness_check',
    COUNTIF(TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), created_at, HOUR) > 24),
    COUNT(*)
  FROM `project.recommendation.user_features`

  UNION ALL

  -- Referential integrity
  SELECT
    'user_product_interactions',
    'product_id',
    'referential_integrity',
    COUNT(*) - COUNT(p.product_id),
    COUNT(*)
  FROM `project.recommendation.user_product_interactions` i
  LEFT JOIN `project.catalog.products` p USING (product_id)
)
SELECT
  *,
  CURRENT_TIMESTAMP() as check_timestamp,
  failures > 0 as has_failures,
  SAFE_DIVIDE(failures, total_rows) * 100 as failure_rate_percent
FROM validation_results;
```

### Statistical Anomaly Detection
```sql
-- Detect anomalies using z-score
WITH stats AS (
  SELECT
    AVG(daily_events) as mean_events,
    STDDEV(daily_events) as stddev_events
  FROM user_daily_metrics
  WHERE event_date >= CURRENT_DATE() - 30
),
z_scores AS (
  SELECT
    user_id,
    event_date,
    daily_events,
    (daily_events - stats.mean_events) / NULLIF(stats.stddev_events, 0) as z_score
  FROM user_daily_metrics
  CROSS JOIN stats
  WHERE event_date = CURRENT_DATE()
)
SELECT
  user_id,
  event_date,
  daily_events,
  z_score,
  CASE
    WHEN ABS(z_score) > 3 THEN 'CRITICAL_ANOMALY'
    WHEN ABS(z_score) > 2 THEN 'WARNING'
    ELSE 'NORMAL'
  END as anomaly_level
FROM z_scores
WHERE ABS(z_score) > 2
ORDER BY ABS(z_score) DESC;
```

## Performance Troubleshooting

### Identify Slow Queries
```sql
-- Find slowest queries in the last 24 hours
SELECT
  job_id,
  user_email,
  TIMESTAMP_DIFF(end_time, start_time, SECOND) as duration_seconds,
  total_slot_ms / 1000 as slot_seconds,
  total_bytes_processed / POW(10, 9) as gb_processed,
  SUBSTR(query, 1, 200) as query_preview,
  -- Performance ratio (lower is better)
  SAFE_DIVIDE(total_slot_ms / 1000, TIMESTAMP_DIFF(end_time, start_time, SECOND)) as parallelism
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND job_type = 'QUERY'
  AND state = 'DONE'
  AND TIMESTAMP_DIFF(end_time, start_time, SECOND) > 60  -- Queries taking > 1 minute
ORDER BY duration_seconds DESC
LIMIT 20;
```

### Optimize Skewed Joins
```sql
-- Problem: Skewed join on user_id
-- Bad approach
SELECT u.user_id, e.event_count
FROM users u
JOIN (
  SELECT user_id, COUNT(*) as event_count
  FROM events
  GROUP BY user_id
) e USING (user_id);

-- Solution: Use approximate aggregation or filter hot keys
SELECT u.user_id, e.event_count
FROM users u
JOIN (
  SELECT user_id, APPROX_COUNT_DISTINCT(event_id) as event_count
  FROM events
  WHERE user_id NOT IN (
    -- Exclude hot keys
    SELECT user_id FROM events
    GROUP BY user_id
    HAVING COUNT(*) > 100000
  )
  GROUP BY user_id
) e USING (user_id);
```

## Best Practices
- Always partition tables by date for time-series data
- Use clustering on frequently filtered columns (user_id, product_id)
- Avoid self-joins - use window functions instead
- Use SAFE_DIVIDE to handle division by zero
- Cache expensive queries with materialized views
- Use approximate aggregations (APPROX_COUNT_DISTINCT) for large datasets
- Analyze query performance with INFORMATION_SCHEMA
- Set up cost monitoring and budget alerts
- Implement comprehensive data quality checks
- Use ML.EXPLAIN_PREDICT for model interpretability
- Leverage incremental training for large models
- Use cyclical encoding for seasonal features
- Monitor and optimize slow queries regularly
- Handle data skew with approximate methods or filtering
