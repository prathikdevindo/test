## query.sql
Language: sql
Total Lines: 47

```sql
## Introduction
-- This SQL script is designed to perform complex data retrieval and transformation operations on a set of biological interaction data. 
-- It utilizes Common Table Expressions (CTEs) to organize and simplify the query structure, making it easier to understand and maintain.

## Installation
-- Ensure that you have access to a SQL database that contains the necessary tables: 
-- `co_bi_inter_conditiongroups`, `co_bi_inter_groupidconditions`, and `co_bi_inter_genebinayinter`.
-- No additional installation steps are required beyond having a compatible SQL environment.

## APIs
-- This script does not directly interact with external APIs. It operates entirely within the SQL database environment.

## Integrations
-- The script integrates data from multiple tables within the database to derive meaningful insights about gene interactions and conditions.

## Utilities
-- The script uses several CTEs to break down the query into manageable parts:
-- 1. `Kin`: Identifies distinct condition IDs and site IDs associated with a specific gene.
-- 2. `inter_binary`: Finds distinct gene IDs that share the same interactor ID.
-- 3. `inter`: Retrieves distinct condition IDs, site IDs, and gene IDs for genes found in `inter_binary`.
-- 4. `count_kin`: Counts the frequency of distinct condition IDs for a specific gene, grouped by site ID.

## Web
-- This script is intended for use in a database environment and does not include web-based components or interfaces.
```
```sql
-- CTE: count_inter
-- This Common Table Expression (CTE) calculates the interaction frequency for each site and gene.
-- It counts the distinct condition IDs associated with each gene and site combination.
-- The data is sourced from the 'co_bi_inter_conditiongroups' and 'co_bi_inter_groupidconditions' tables.
-- The CTE filters genes that are present in the 'inter_binary' table.
count_inter AS (
    SELECT 
        COUNT(DISTINCT co_bi_inter_groupidconditions.condition_id_id) AS interact_freq,
        site_id,
        gene_id
    FROM 
        co_bi_inter_conditiongroups 
    JOIN 
        co_bi_inter_groupidconditions 
    ON 
        co_bi_inter_conditiongroups.condition_group_id = co_bi_inter_groupidconditions.groupid_id 
    WHERE 
        co_bi_inter_conditiongroups.gene_id IN (SELECT * FROM inter_binary) 
    GROUP BY 
        site_id, gene_id
),

-- CTE: shared_freq
-- This CTE calculates the shared frequency of conditions between 'kin' and 'inter' tables.
-- It counts the distinct condition IDs shared between the two tables for each site and gene.
-- The data is grouped by 'kin_sit', 'kin', 'site_id', and 'gene_id'.
shared_freq AS (
    SELECT 
        kin,
        kin_sit,
        COUNT(DISTINCT kin.condition_id_id) AS shared_freqn,
        gene_id AS Binary_interactor,
        site_id
    FROM 
        kin 
    INNER JOIN 
        inter 
    ON 
        kin.condition_id_id = inter.condition_id_id 
    GROUP BY 
        kin_sit, kin, site_id, gene_id
)

-- Main Query
-- This query selects and calculates various metrics related to shared frequencies and interaction frequencies.
-- It joins the 'shared_freq' CTE with 'count_kin' and 'count_inter' to obtain necessary frequency data.
-- The query calculates the minimum frequency between 'kin_freq' and 'interact_freq' and computes a 'shared_score'.
-- The 'shared_score' is the ratio of 'shared_freqn' to the minimum frequency, cast to a decimal for precision.
SELECT 
    shared_freq.kin,
    kin_sit,
    kin_freq,
    shared_freqn,
    Binary_interactor,
    shared_freq.site_id,
    interact_freq,
    LEAST(kin_freq, interact_freq) AS min_freq,
    (CAST(shared_freqn AS DECIMAL(10, 2)) / CAST(LEAST(kin_freq, interact_freq) AS DECIMAL(10, 2))) AS shared_score
FROM 
    shared_freq 
JOIN 
    count_kin 
ON 
    count_kin.site_id = shared_freq.kin_sit 
JOIN 
    count_inter 
ON 
    shared_freq.Binary_interactor = count_inter.gene_id 
AND 
    shared_freq.site_id = count_inter.site_id;
```

---
📄 Generated on 2026-01-08
