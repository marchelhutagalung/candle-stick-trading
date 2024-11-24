# High-Throughput Trading Data System Design
A scalable system design for processing real-time trading data and generating candlestick charts.

## Table of Contents
1. [Core Requirements](#core-requirements)
2. [System Architecture](#system-architecture)
3. [Data Model](#data-model)
4. [Performance Optimization](#performance-optimization)
5. [Query Patterns](#query-patterns)
6. [Trade-offs](#trade-offs)
7. [Assumptions](#assumptions)

## Core Requirements
- Process real-time trade data
- Generate candlestick charts (HLOC) within intervals (30min, 1hr, 4hr, 1day)
- Scale up to terabytes of historical data
- Support fast data retrieval

### Input Data Format
```json
{
    "ID": 110574616,
    "Totaldone": 500000.0,
    "Trx Amount": 4.5115767058272E-4,
    "Amount": 1.10826E9,
    "Transtype": 1,
    "Accountid": 1,
    "Transtime": "2024-11-05T22:02:53Z"
}
```

## System Architecture

### Technology Stack
- **Language: Go** - For high performance and excellent concurrency support
- **Message Queue: Kafka** - For reliable high-throughput data ingestion
- **Cache: Redis** - For fast in-memory data access
- **Database: TimescaleDB** - For optimized time-series data storage
- **API: REST** - For standardized and scalable data access

### System Components
[![](https://mermaid.ink/img/pako:eNp9VNtu2zAM_RXCzw2wrHnKw4DEbt0t3eI6QQJMDgbOpm2hthTI8oqi7r9XviXOZdWLKPEc8pAi9GaFMiJrasWZfAlTVBrWTiDArKL8myjcp_Bd7EsNrN0e8ZXUrkXUa-2wtcKIwEGNOxiNvlU_VstfFSzu2QLjZxxgF_e1H6qVVoR5BRuPbTDjEWouBXhKhlQUchh947WMBtakqGDusXkZx6TAkzLrwCSiQJzp7gJykQAb2OcVzLscc9RhWsHMZbMkUZSgPpEyc1vYw-PSrsB2mY0iyqjQPHwGlwSpM4LdEdY8J1iR4lRU4GxZXQVsFdcHFdfEr0wwTAhYb5zLdrZteJ8wG2mTowLfYT5FvLiGwpeufevVH3-2ZbWqIsSMnDkY5xVK3wWKGtLMdU9IR_eAe7QOhTSKwMYwpaO3Xr7T5rn9knNhOjpmjdVCd9exY_NA9lc2Tj9FTWrULZt8jhqbuuwJG0cXqMF7nJQyKP80ZNvTNqwt870yo9Y0zV7-9Fh_Q1H_qrsLuulu13UVpvwftS337QfWXVyhNjL_Nz9PJalXYO12PjtPd53jTiRc0K6fpJCErsdoiOymnhcmPzfF9xN0ibkcmIM868bKSeXII_PVvNXXgaVTyimwpsaMKMYy04EViHcDxVLL1asIralWJd1YSpZJ2h_KvfkvyOFo6sytaYxZYW73KH5L2Z_fPwBH_HXw?type=png)](https://mermaid.live/edit#pako:eNp9VNtu2zAM_RXCzw2wrHnKw4DEbt0t3eI6QQJMDgbOpm2hthTI8oqi7r9XviXOZdWLKPEc8pAi9GaFMiJrasWZfAlTVBrWTiDArKL8myjcp_Bd7EsNrN0e8ZXUrkXUa-2wtcKIwEGNOxiNvlU_VstfFSzu2QLjZxxgF_e1H6qVVoR5BRuPbTDjEWouBXhKhlQUchh947WMBtakqGDusXkZx6TAkzLrwCSiQJzp7gJykQAb2OcVzLscc9RhWsHMZbMkUZSgPpEyc1vYw-PSrsB2mY0iyqjQPHwGlwSpM4LdEdY8J1iR4lRU4GxZXQVsFdcHFdfEr0wwTAhYb5zLdrZteJ8wG2mTowLfYT5FvLiGwpeufevVH3-2ZbWqIsSMnDkY5xVK3wWKGtLMdU9IR_eAe7QOhTSKwMYwpaO3Xr7T5rn9knNhOjpmjdVCd9exY_NA9lc2Tj9FTWrULZt8jhqbuuwJG0cXqMF7nJQyKP80ZNvTNqwt870yo9Y0zV7-9Fh_Q1H_qrsLuulu13UVpvwftS337QfWXVyhNjL_Nz9PJalXYO12PjtPd53jTiRc0K6fpJCErsdoiOymnhcmPzfF9xN0ibkcmIM868bKSeXII_PVvNXXgaVTyimwpsaMKMYy04EViHcDxVLL1asIralWJd1YSpZJ2h_KvfkvyOFo6sytaYxZYW73KH5L2Z_fPwBH_HXw)

## Data Model

### Database Schema
```sql
-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Raw trades table
CREATE TABLE trades (
    id BIGINT NOT NULL,
    total_done DECIMAL(20,2) NOT NULL,
    trx_amount DECIMAL(20,8) NOT NULL,
    amount DECIMAL(20,2) NOT NULL,
    trans_type SMALLINT NOT NULL,
    account_id INTEGER NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    
    PRIMARY KEY (timestamp, id)
);

-- Convert to hypertable
SELECT create_hypertable('trades', 'timestamp', 
    chunk_time_interval => INTERVAL '1 day');

-- Candlestick table
CREATE TABLE candlesticks (
    interval_start TIMESTAMPTZ NOT NULL,
    interval_type VARCHAR(10) NOT NULL,
    account_id INTEGER NOT NULL,
    open_price DECIMAL(20,2) NOT NULL,
    high_price DECIMAL(20,2) NOT NULL,
    low_price DECIMAL(20,2) NOT NULL,
    close_price DECIMAL(20,2) NOT NULL,
    volume DECIMAL(20,8) NOT NULL,
    
    PRIMARY KEY (interval_start, interval_type, account_id)
);

-- Convert to hypertable
SELECT create_hypertable('candlesticks', 'interval_start');
```

### Indexing Strategy
```sql
-- Optimize time-range queries
CREATE INDEX idx_trades_time ON trades USING BRIN (timestamp);

-- Optimize account-specific queries
CREATE INDEX idx_trades_account ON trades (account_id, timestamp DESC);

-- Optimize candlestick retrieval
CREATE INDEX idx_candlesticks_lookup 
ON candlesticks (account_id, interval_type, interval_start DESC);
```

### Continuous Aggregates
```sql
CREATE MATERIALIZED VIEW candlesticks_30m
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('30 minutes', timestamp) AS interval_start,
    account_id,
    first(amount, timestamp) as open_price,
    max(amount) as high_price,
    min(amount) as low_price,
    last(amount, timestamp) as close_price,
    sum(trx_amount) as volume
FROM trades
GROUP BY time_bucket('30 minutes', timestamp), account_id;
```

## Performance Optimization

### Data Compression
```sql
-- Enable compression
ALTER TABLE trades SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'account_id'
);

-- Add compression policy
SELECT add_compression_policy('trades', INTERVAL '7 days');
```

### Retention Policies
```sql
-- Add retention policy
SELECT add_retention_policy('trades', INTERVAL '30 days');
SELECT add_retention_policy('candlesticks', INTERVAL '7 years');
```

### Resource Management
```sql
-- Configure chunk size
SELECT set_chunk_time_interval('trades', INTERVAL '1 day');

-- Configure memory
ALTER DATABASE trading SET timescaledb.max_memory_size = '4GB';
```

## Query Patterns

### Recent Data Access
```sql
-- Get latest trades
SELECT * FROM trades
WHERE account_id = $1 
AND timestamp >= NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;

-- Get latest candlestick
SELECT * FROM candlesticks
WHERE account_id = $1
AND interval_type = '30m'
AND interval_start >= NOW() - INTERVAL '30 minutes';
```

### Historical Data Access
```sql
-- Get historical candlesticks
SELECT * FROM candlesticks
WHERE account_id = $1
AND interval_type = $2
AND interval_start BETWEEN $3 AND $4
ORDER BY interval_start;
```

## Trade-offs

### Storage vs Performance
- Hot data: In-memory and cached
- Warm data: Hot partition
- Cold data: Compressed storage

### Consistency vs Latency
- Real-time data: Eventually consistent
- Historical data: Strongly consistent
- Automated reconciliation process

## Assumptions
- Peak throughput: 10,000 TPS
- Data retention: Hot (30 days), Cold (7 years)
- Query patterns: 80% recent, 20% historical
- Maximum acceptable latency: 100ms for recent data

