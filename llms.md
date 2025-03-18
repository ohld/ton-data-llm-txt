# TON Blockchain data analysis with Dune

This document cointains the guide for AI agents to understand how to write Dune SQL queries and analyse TON Blockchain data. 

# Facts about Dune you should know
To write a query on Dune, you need to follow Presto SQL syntax.


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
        -- SUM(ABS(amount_) * P.price_usd) AS volume_usd,
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


## Labels

### Labels from github repo 
```sql
SELECT address, label, organization, category
FROM dune.ton_foundation.dataset_labels
```

### NFT and NFT collections names

```sql
WITH
NFT_NAMES AS (
    SELECT 
        address, 
        MAX(parent_address) AS parent_address,
        MAX_BY(
            COALESCE(
                name,
                json_extract_scalar(content_onchain, '$.domain') || '.ton'
            )
            , adding_date
        ) AS name
    FROM ton.nft_metadata NM
    GROUP BY 1
)
, NFT_COLLECTION_NAME AS (
    -- in case you want to aggregate NFTs by collection name if presented
    SELECT 
        NFT_NAMES.address, COALESCE(COLLECTION.name, NFT_NAMES.name) AS name
    FROM NFT_NAMES
    LEFT JOIN NFT_NAMES AS COLLECTION
        ON COLLECTION.address = NFT_NAMES.parent_address
    WHERE COALESCE(COLLECTION.name, NFT_NAMES.name) IS NOT NULL 
)
```


## Misc

Useful links:
1. https://docs.ton.org/v3/documentation/data-formats/tlb/cell-boc
2. https://blog.ton.org/ton-on-chain-data-analysis-dune
