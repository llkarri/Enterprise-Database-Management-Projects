# 📊 USDT Competitive Intelligence Dashboard
**[View Dashboard](https://dune.com/eugenielai_team_3740/badm-554-group2-miniproject2)**
## Overview
A cross-token intelligence dashboard comparing USDT against USDC, DAI, and PYUSD across volume trends, sector penetration, user profiles, and retention behavior. 

This dashboard answers one question: **_Where is USDT winning, and where is it losing ground?_**
All charts are interactive, ensuring dynamic and real-time competitive insights.

## Purpose & Business Questions

This dashboard addresses the following key business questions:

1. **Market Share & Growth:** Where is USDT winning and where is it losing ground vs. USDC, DAI, and PYUSD?
2. **Sector Penetration:** Which blockchain sectors (DeFi, CEX, Bridges, MEV, P2P) drive inflows for each token?
3. **User Composition:** What is the mix of retail, trader, and whale volume for USDT vs. competitors?
4. **Retention & Stickiness:** How loyal are users? What % remain active for 30+ days?
5. **Volume Trends:** How do daily transfer volumes compare across tokens over time?

---

## USDT 60-Day Headlines
Use this for: Counters (Volume, Tx Count, Active Users)
This query calculates USDT’s headline activity metrics over the past 60 days, including total volume, transaction count, and the number of unique active wallets.
```sql
SELECT 
    -- Counter 1: Volume in Billions 
    -- Logic: (Value / 6 decimals) / 1,000,000,000
    SUM(value / 1e6) / 1e9 as total_volume_billions,
    
    -- Counter 2: Total Number of Transfers
    COUNT(*) as total_transfers,
    
    -- Counter 3: Unique Active Wallets (People)
    COUNT(DISTINCT "from") as unique_active_users

FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 -- USDT Only
AND evt_block_time > NOW() - INTERVAL '60' DAY
```
## USDT Long-Term Retention Counter
Logic: Calculates % of users who hold > 30 Days.
Generates long-term retention rates (% of 30+ day active users) for major stablecoins based on historical transfer activity.
```sql
WITH raw AS (
    SELECT
        contract_address,
        "from" AS wallet,
        evt_block_time
    FROM erc20_ethereum.evt_Transfer
    WHERE contract_address IN (
        0xdac17f958d2ee523a2206206994597c13d831ec7, -- USDT
        0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48, -- USDC
        0x6b175474e89094c44da98b954eedeac495271d0f, -- DAI
        0x6c3ea9036406852006290770bedfcaba0e23a0e8  -- PYUSD
    )
    AND evt_block_time > NOW() - INTERVAL '180' DAY
),

lifecycle AS (
    SELECT
        CASE 
            WHEN contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 THEN 'USDT'
            WHEN contract_address = 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 THEN 'USDC'
            WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN 'DAI'
            ELSE 'PYUSD'
        END AS token,
        DATE_DIFF('day', MIN(evt_block_time), MAX(evt_block_time)) AS retention_days
    FROM raw
    GROUP BY 1, wallet
)

SELECT
    -- This calculates the % of users with >30 days retention for each token
    COUNT(CASE WHEN token = 'USDT' AND retention_days > 30 THEN 1 END) * 100.0 / NULLIF(COUNT(CASE WHEN token = 'USDT' THEN 1 END), 0) as usdt_loyal_pct,
    
    COUNT(CASE WHEN token = 'USDC' AND retention_days > 30 THEN 1 END) * 100.0 / NULLIF(COUNT(CASE WHEN token = 'USDC' THEN 1 END), 0) as usdc_loyal_pct,
    
    COUNT(CASE WHEN token = 'DAI' AND retention_days > 30 THEN 1 END) * 100.0 / NULLIF(COUNT(CASE WHEN token = 'DAI' THEN 1 END), 0) as dai_loyal_pct
    FROM lifecycle
```
## The "Market Evolution" Chart (Volume Trends)
This query computes daily transfer volume (USD-normalized) for USDT, USDC, DAI, and PYUSD over the past 60 days.
```sql
WITH raw_data AS (
    SELECT
        date_trunc('day', evt_block_time) as time,
        CASE
            WHEN contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 THEN 'USDT'
            WHEN contract_address = 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 THEN 'USDC'
            WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN 'DAI'
            WHEN contract_address = 0x6c3ea9036406852006290770bedfcaba0e23a0e8 THEN 'PYUSD'
        END as symbol,
        CASE
            WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN value / 1e18
            ELSE value / 1e6
        END as amount_usd
    FROM erc20_ethereum.evt_Transfer
    WHERE contract_address IN (
        0xdac17f958d2ee523a2206206994597c13d831ec7, -- USDT
        0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48, -- USDC
        0x6b175474e89094c44da98b954eedeac495271d0f, -- DAI
        0x6c3ea9036406852006290770bedfcaba0e23a0e8  -- PYUSD
    )
    AND evt_block_time > NOW() - INTERVAL '60' day
)
SELECT
    time,
    symbol,
    SUM(amount_usd) as daily_volume
FROM raw_data
WHERE symbol IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC;
```
## Where is the money (Sector Penetration)
This query classifies 60-day stablecoin inflows into major ecosystem sectors (DeFi, CEX, Bridges, MEV, and unlabeled wallets) and aggregates their USD volume.
```sql
WITH raw_data AS (
 SELECT
 CASE
 WHEN t.contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 
THEN 'USDT'
 WHEN t.contract_address = 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 
THEN 'USDC'
 WHEN t.contract_address = 0x6c3ea9036406852006290770bedfcaba0e23a0e8 
THEN 'PYUSD'
 WHEN t.contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f 
THEN 'DAI'
 END as token,
 l.category,
 -- COMPLEX MATH: Handle DAI's 18 decimals vs others' 6 decimals
 CASE
 WHEN t.contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f 
THEN t.value / 1e18
 ELSE t.value / 1e6
 END as amount_usd
 FROM erc20_ethereum.evt_Transfer t
 -- Using labels.addresses as per your working snippet
 LEFT JOIN labels.addresses l ON t."to" = l.address AND l.blockchain = 'ethereum'
 WHERE t.contract_address IN (
 0xdac17f958d2ee523a2206206994597c13d831ec7, -- USDT
 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48, -- USDC
 0x6c3ea9036406852006290770bedfcaba0e23a0e8, -- PYUSD
 0x6b175474e89094c44da98b954eedeac495271d0f -- DAI
 )
 AND t.evt_block_time > NOW() - INTERVAL '60' day
)
SELECT
 token,
 -- CLEAN UP: Group the hundreds of labels into 5 Business Sectors
 CASE
 WHEN category IN ('dex', 'amm', 'balancer', 'uniswap', 'sushiswap', 'curve', '1inch') THEN 
'1. DeFi / DEX'
 WHEN category IN ('cex', 'exchange') THEN '2. CEX (Centralized Exchange)'
 WHEN category IN ('bridge', 'layer2', 'optimism', 'arbitrum') THEN '3. Bridge / L2'
 WHEN category IN ('mev', 'bot', 'operator') THEN '4. MEV Bots'
 WHEN category IS NULL THEN '5. Unlabeled / P2P'
 ELSE '6. Other Protocols'
 END as sector_group,
 SUM(amount_usd) as volume_usd
FROM raw_data
-- Filter out small transfers (<$1000) here in the outer query
WHERE amount_usd > 1000
GROUP BY 1, 2
HAVING SUM(amount_usd) > 1000000 -- Remove tiny noise sectors
ORDER BY 1, 2
```

## User Behavior in Total Volume
This query segments 60-day stablecoin transfers into retail, trader, and whale groups based on transfer size, and aggregates their frequency and total volume.
```sql
WITH transfer_data AS (
 SELECT
 CASE
 WHEN contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 THEN 
'USDT'
 WHEN contract_address = 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 
THEN 'USDC'
 WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN 
'DAI'
 WHEN contract_address = 0x6c3ea9036406852006290770bedfcaba0e23a0e8 THEN 
'PYUSD'
 END as token,
 -- COMPLEX MATH: Handle decimals correctly
 CASE
 -- DAI uses 18 decimals
 WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN 
value / 1e18
 -- Everyone else (USDT, USDC, PYUSD) uses 6
 ELSE value / 1e6
 END as amount_usd
 FROM erc20_ethereum.evt_Transfer
 WHERE contract_address IN (
 0xdac17f958d2ee523a2206206994597c13d831ec7, -- USDT
 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48, -- USDC
 0x6b175474e89094c44da98b954eedeac495271d0f, -- DAI
 0x6c3ea9036406852006290770bedfcaba0e23a0e8 -- PYUSD
 )
 AND evt_block_time > NOW() - INTERVAL '60' day
 -- Optimization: Filter out 0 value transfers to speed up query
 AND value > 0
)
SELECT
 token,
 CASE
 WHEN amount_usd < 1000 THEN '1. Retail (<$1k)'
 WHEN amount_usd >= 1000 AND amount_usd < 100000 THEN '2. Trader ($1k-$100k)'
 WHEN amount_usd >= 100000 THEN '3. Whale (>$100k)'
 END as user_type,
 COUNT(*) as transfer_count,
 SUM(amount_usd) as total_volume
FROM transfer_data
WHERE token IS NOT NULL
GROUP BY 1, 2
ORDER BY 1, 2
```

## Token Retention (Stickiness)
This query builds lifecycle buckets for each stablecoin by measuring how many days each wallet remained active over the past 180 days, then counts users in each bucket.
```sql
WITH raw AS (
    SELECT
        contract_address,
        "from" AS wallet,
        evt_block_time
    FROM erc20_ethereum.evt_Transfer
    WHERE contract_address IN (
        0xdac17f958d2ee523a2206206994597c13d831ec7, -- USDT
        0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48, -- USDC
        0x6b175474e89094c44da98b954eedeac495271d0f, -- DAI
        0x6c3ea9036406852006290770bedfcaba0e23a0e8  -- PYUSD
    )
    AND evt_block_time > NOW() - INTERVAL '180' DAY
),

lifecycle AS (
    SELECT
        contract_address,
        wallet,
        MIN(evt_block_time) AS first_seen,
        MAX(evt_block_time) AS last_seen,
        DATE_DIFF('day', MIN(evt_block_time), MAX(evt_block_time)) AS retention_days
    FROM raw
    GROUP BY 1, 2
),

bucketed AS (
    SELECT
        CASE 
            WHEN contract_address = 0xdac17f958d2ee523a2206206994597c13d831ec7 THEN 'USDT'
            WHEN contract_address = 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 THEN 'USDC'
            WHEN contract_address = 0x6b175474e89094c44da98b954eedeac495271d0f THEN 'DAI'
            WHEN contract_address = 0x6c3ea9036406852006290770bedfcaba0e23a0e8 THEN 'PYUSD'
        END AS token,

        CASE
            WHEN retention_days = 0 THEN '0-day'
            WHEN retention_days BETWEEN 1 AND 7 THEN '1–7 days'
            WHEN retention_days BETWEEN 8 AND 30 THEN '7–30 days'
            WHEN retention_days > 30 THEN '30+ days'
        END AS bucket
    FROM lifecycle
)

SELECT
    token,
    bucket,
    COUNT(*) AS user_count
FROM bucketed
GROUP BY 1, 2
ORDER BY token, bucket;
```