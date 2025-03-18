Useful links:
1. https://docs.ton.org/v3/documentation/data-formats/tlb/cell-boc
2. https://blog.ton.org/ton-on-chain-data-analysis-dune



# Useful Dune snippets
For now, I just copy-paste some useful snippets that I use

## Calculate net value flow between addresses

```sql
WITH JETTON_TRANSFERS AS (
    SELECT 
        block_date, source, destination, jetton_master AS token_address, amount
    FROM ton.jetton_events JE
    WHERE 1=1
        AND tx_aborted = FALSE
        AND block_date >= TIMESTAMP '2024-12-20' -- it's important to select only partitions you need
        AND amount > 0
        AND source IS NOT NULL AND destination IS NOT NULL -- not sure about that
)

, TON_TRANSFERS AS (
    SELECT 
        block_date, source, destination, 
        '0:0000000000000000000000000000000000000000000000000000000000000000' AS token_address, -- address of TON token
        value AS amount
    FROM ton.messages M
    WHERE 1=1
        AND direction = 'in'
        AND bounced = FALSE 
        AND value > POWER(10, 9 + 1)  -- at least 10 ton transfer
        AND block_date >= TIMESTAMP '2024-12-20'  -- it's important to select only partitions you need
)

, TRANSFERS AS (
    SELECT * FROM JETTON_TRANSFERS
    UNION ALL
    SELECT * FROM TON_TRANSFERS
)

, NET_VOLUME AS (
    SELECT 
        from_,
        to_,
        SUM(amount_ * P.price_usd) AS net_volume_usd
    FROM TRANSFERS T
    CROSS JOIN UNNEST (ARRAY[
      ROW(source, destination, amount), 
      ROW(destination, source, amount * -1)
    ]) AS T(from_, to_, amount_)
    INNER JOIN ton.prices_daily P
        ON P.timestamp = T.block_date
        AND P.token_address = T.token_address
    GROUP BY 1,2
)
```

In result, you get the table that shows the value flow between addresses. 