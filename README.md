# Comm-Log Send Reconciliation

## 1. Reconciliation Bridge

| Step | Description | Result | Reason |
|---|---|---:|---|
| 0 | Naive count of all `communication_log` rows | **30** | The most obvious interpretation is to treat every log row as a send attempt. |
| 1 | Filter to campaigns eligible for official reporting | **26** | Campaign 9004 has `creation_status = 'approval_awaiting'`. Its 4 communication-log rows exist, but the README explicitly says such campaigns are not reportable. |
| 2 | Filter to successfully delivered attempts (`delivery_status = 900`) | **22** | `900` represents a successfully delivered send. The 26 reportable rows contain 22 successful and 4 failed attempts. |
| 3 | Validate retry-chain semantics | **22** | The README defines `target_base` at the underlying communication level: customers reached through multiple attempts in the same retry chain count once. In this dataset, no customer has more than one successful delivery within a retry chain, so the 22 delivered rows remain 22 reached customers/events. |
| 4 | Validate standalone semantics | **22** | Campaign 9101 is standalone. Its 7 delivered sends are therefore 7 independent events, including the two sends to C20. |
| **Final** | **Finance `target_base`** | **22** | 10 customers from 9001→9002→9003, 5 customers from 9201→9202, and 7 standalone events from 9101. |

### Final reconciliation

**10 + 5 + 7 = 22**

## 2. SQL Used to Compute the Final Number

Query 1 — Initial straightforward approach

```sql
SELECT COUNT(*) AS target_base
FROM communication_log cl
JOIN campaign c
    ON cl.communication_id = c.id
WHERE c.creation_status IN ('approved', 'aborted', 'resumed', 'stopped')
  AND c.processing_status = 'processed'
  AND cl.delivery_status = 900;
```
This was the initial straightforward interpretation: count successfully delivered sends from campaigns that are eligible for official reporting.

After reviewing the README more closely, I validated this result against the definition of target_base, which treats retries as part of the same underlying communication and treats standalone campaigns differently.

```sql
WITH RECURSIVE chains AS (
    SELECT
        id AS campaign_id,
        id AS root_id
    FROM campaign
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.id,
        chains.root_id
    FROM campaign c
    JOIN chains
        ON c.parent_id = chains.campaign_id
),

sends AS (
    SELECT
        chains.root_id,
        cl.id AS send_id,
        cl.customer_id
    FROM communication_log cl
    JOIN chains
        ON cl.communication_id = chains.campaign_id
    JOIN campaign c
        ON c.id = cl.communication_id
    WHERE c.creation_status IN ('approved', 'aborted', 'resumed', 'stopped')
      AND c.processing_status = 'processed'
      AND cl.delivery_status = 900
),

counts AS (
    SELECT
        root_id,
        COUNT(DISTINCT customer_id) AS target_base
    FROM sends
    WHERE root_id IN (9001, 9201)
    GROUP BY root_id

    UNION ALL

    SELECT
        root_id,
        COUNT(*) AS target_base
    FROM sends
    WHERE root_id = 9101
    GROUP BY root_id
)

SELECT SUM(target_base) AS target_base
FROM counts;

```

The above query implements the Finance definition: retry chains count distinct successfully reached customers, while standalone communications count every send as a separate event.

### Output

```text
target_base
-----------
22
```

## 3. What Surprised Me

`C20` appears twice under standalone campaign `9101`, but both correctly count as separate events. The simpler delivered-send query also happened to return 22 because no customer had multiple successful deliveries within a retry chain. Additionally, campaign `9004` had four sends despite still being `approval_awaiting`, so those rows were excluded from reporting.

## 4. Analysis Methodology

The analysis was performed sequentially in `analytics.ipynb` using Python and SQLite. The notebook connects to `comm_log.db` and uses `pandas.read_sql_query()` to execute and inspect each SQL query step by step, from initial data exploration through the final reconciliation.
