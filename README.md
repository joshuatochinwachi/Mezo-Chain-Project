# What's really going on with Mezo (testnet) chain?
[Full Dashboard](https://joshuatochinwachi.github.io/Mezo-Chain-Project) | [Data story telling on 𝕏](https://x.com/defi__josh/status/1924202448203190390)

This dashboard and entire analysis was built and developed using [Flipside Crypto](https://flipsidecrypto.xyz) by me. Unfortunately, Flipside’s querying and dashboarding tool has gone dark because of their collaboration with [Snowflake](https://www.snowflake.com/en/).
Flipside also used Snowflake’s SQL dialect as their SQL flavor when they were active, so the SQL code you’ll see in this project—for both querying and dashboarding—is very similar to Snowflake’s syntax.

Using [Python scripts](https://github.com/joshuatochinwachi/Flipside_dashboard_porter), I scraped my Flipside dashboard and visualizations from the Flipside website.

## Queries/Metrics used

### 1. Mezo Chain Launch Date and First Activity Metrics 

```
WITH daily_stats AS (
    SELECT 
        DATE_TRUNC('day', block_timestamp) as date,
        MIN(block_timestamp) as first_block_time,
        MIN(block_number) as first_block_number,
        COUNT(DISTINCT block_number) as blocks_created,
        COUNT(DISTINCT tx_hash) as total_transactions,
        AVG(gas_used) as avg_gas_used,
        AVG(tx_fee) as avg_tx_fee
    FROM mezo.testnet.fact_transactions
    GROUP BY 1
    ORDER BY 1
),

chain_stats AS (
    SELECT 
        MIN(date) as "chain launch date",
        first_block_time as "first block and transaction time",
        first_block_number as "first block number",
        blocks_created as "no. of blocks created on first day",
        total_transactions as "no. of transactions on first day",
        ROUND(avg_tx_fee, 6) as "first day avg transaction fee"
    FROM daily_stats
    WHERE date = (SELECT MIN(date) FROM daily_stats)
    GROUP BY 2, first_block_number, blocks_created, total_transactions, avg_tx_fee
)

SELECT 
    c.*,
    t.tx_hash as "first transaction hash",
    t.from_address as "first user to transact on Mezo chain (from address)",
    t.to_address as "first recipient (to address)",
    t.value as "value transacted",
    t.tx_succeeded as "succeeded?"
FROM chain_stats c
CROSS JOIN mezo.testnet.fact_transactions t
WHERE t.block_timestamp = c."first block and transaction time"
ORDER BY t.block_number, t.tx_position;
```

### 2. Comprehensive Daily Network Metrics (since launch)

```
WITH daily_stats AS (
    SELECT 
        DATE_TRUNC('day', block_timestamp) as date,
        COUNT(DISTINCT block_number) as blocks_created,
        COUNT(DISTINCT tx_hash) as total_transactions,
        COUNT(DISTINCT from_address) as unique_active_addresses,
        AVG(gas_used) as avg_gas_used,
        MAX(gas_used) as max_gas_used,
        MIN(gas_used) as min_gas_used,
        AVG(tx_fee) as avg_tx_fee,
        SUM(tx_fee) as total_tx_fees,
        SUM(CASE WHEN tx_succeeded = true THEN 1 ELSE 0 END) as successful_tx,
        ROUND(SUM(CASE WHEN tx_succeeded = true THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as success_rate,
        SUM(value) as total_value_transferred
    FROM mezo.testnet.fact_transactions
    WHERE block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
    GROUP BY 1
    ORDER BY 1
)

SELECT 
    date as "day",
    blocks_created as "blocks created",
    SUM(blocks_created) OVER (ORDER BY date) AS "cumulative blocks created",
    total_transactions as "total transactions",
    SUM(total_transactions) OVER (ORDER BY date) AS "cumulative total transactions",
    unique_active_addresses as "unique active addresses",
    SUM(unique_active_addresses) OVER (ORDER BY date) AS "cumulative unique active addresses",
    successful_tx as "successful transactions",
    SUM(successful_tx) OVER (ORDER BY date) AS "cumulative successful transactions",
    success_rate as "success rate %",
    ROUND(avg_gas_used, 2) as "avg gas used",
    max_gas_used as "max gas used",
    min_gas_used as "min gas used",
    ROUND(avg_tx_fee, 6) as "avg tx fee",
    ROUND(total_tx_fees, 6) as "total tx fees",
    ROUND(total_value_transferred, 6) as "total value transferred"
FROM daily_stats
ORDER BY date DESC;
```

### 3. No. of Deployed Contracts and Contract Creators

```
SELECT 
    count(distinct address) as "No. of deployed contracts",
    count(distinct creator_address) as "No. of contract creators"
FROM mezo.testnet.dim_contracts 
WHERE created_block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
)
```

### 4. Daily Network Activity Metrics (since launch)

```
WITH daily_stats AS (
    SELECT 
        DATE_TRUNC('day', block_timestamp) as date,
        COUNT(DISTINCT block_number) as blocks_created,
        COUNT(DISTINCT tx_hash) as total_transactions,
        AVG(gas_used) as avg_gas_used,
        AVG(tx_fee) as avg_tx_fee
    FROM mezo.testnet.fact_transactions
    WHERE block_timestamp >=  (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
    GROUP BY 1
    ORDER BY 1
)

SELECT 
    date as "day",
    blocks_created as "no. of blocks created",
    total_transactions as "no. of tx per day",
    ROUND(avg_tx_fee, 6) as "avg tx fee",
    ROUND(avg_gas_used, 2) as "avg gas used",
FROM daily_stats;
```

### 5. Block Production Metrics (Network performance through block times)

```
WITH block_times AS (
    SELECT 
        block_number,
        block_timestamp,
        LAG(block_timestamp) OVER (ORDER BY block_number) as prev_block_timestamp
    FROM mezo.testnet.fact_blocks
    WHERE block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
)
SELECT 
    DATE_TRUNC('hour', block_timestamp) as "hour",
    COUNT(*) as "blocks produced",
    AVG(DATEDIFF('second', prev_block_timestamp, block_timestamp)) as "avg block time (seconds)",
    MIN(DATEDIFF('second', prev_block_timestamp, block_timestamp)) as "min block time (seconds)",
    MAX(DATEDIFF('second', prev_block_timestamp, block_timestamp)) as "max block time (seconds)"
FROM block_times
WHERE prev_block_timestamp IS NOT NULL
GROUP BY 1
ORDER BY 1;
```

### 6. Network reliability (transaction success rates)

```
SELECT 
    DATE_TRUNC('day', block_timestamp) as "date",
    COUNT(*) as "total transactions",
    SUM(CASE WHEN tx_succeeded = true THEN 1 ELSE 0 END) as "successful transactions",
    ROUND(SUM(CASE WHEN tx_succeeded = true THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as "success rate percentage"
FROM mezo.testnet.fact_transactions
WHERE block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
GROUP BY 1
ORDER BY 1;
```

### 7. Most Active Contracts and usage

```
SELECT 
    c.address as "contract address",
    --c.created_block_timestamp as "deployment date",
    c.creator_address as "creator address",
    COUNT(DISTINCT t.tx_hash) as "total interactions"
FROM mezo.testnet.dim_contracts c
LEFT JOIN mezo.testnet.fact_transactions t 
    ON c.address = t.to_address
WHERE c.created_block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
GROUP BY 1,2
ORDER BY 3 DESC
LIMIT 5;
```

### 8. Most Active Contracts and deployment dates

```
WITH top_contracts AS (
    SELECT 
        c.address as contract_address,
        COUNT(DISTINCT t.tx_hash) as total_interactions
    FROM mezo.testnet.dim_contracts c
    LEFT JOIN mezo.testnet.fact_transactions t 
        ON c.address = t.to_address
    GROUP BY 1
    ORDER BY 2 DESC
    LIMIT 5
)
SELECT 
    tc.contract_address as "contract address",
    DATE(c.created_block_timestamp) as "deployment date"
FROM top_contracts tc
JOIN mezo.testnet.dim_contracts c 
    ON tc.contract_address = c.address
ORDER BY tc.total_interactions DESC;
```

### 9. Most Common Events and interactions

```
---Popular smart contract interactions

SELECT 
    topic_0 as "event signature hash (like Transfer, Approval, Swap, etc.)",
    COUNT(*) as "no. of emissions across all contracts and transactions",
    COUNT(DISTINCT tx_hash) as "no. of unique transactions",
    COUNT(DISTINCT contract_address) as "no. of unique contracts"
FROM mezo.testnet.fact_event_logs
WHERE block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5;
```

### 10. Most active users and number of interactions

```
--User activity patterns

SELECT 
    from_address as "user",
    COUNT(DISTINCT tx_hash) as "total interactions",
    COUNT(DISTINCT DATE_TRUNC('day', block_timestamp)) as "active days",
    SUM(CASE WHEN tx_succeeded = true THEN 1 ELSE 0 END) as "successful transactions",
    AVG(gas_used) as "avg gas used",
    SUM(value) as "total value transferred"
FROM mezo.testnet.fact_transactions
WHERE block_timestamp >= (
                  SELECT MIN(block_timestamp)
                  FROM mezo.testnet.fact_transactions
            )
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5;
```
