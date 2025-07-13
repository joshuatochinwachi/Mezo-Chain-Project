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
