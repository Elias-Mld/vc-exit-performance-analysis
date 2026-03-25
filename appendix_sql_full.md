# Annexe SQL complète (ADD#2)

Ce fichier reprend **les mêmes requêtes** que l’annexe PDF du rapport et que la liste `sql_blocks` dans [`generate_pdf.py`](generate_pdf.py).

**Prérequis :** charger les CSV CrunchBase dans SQLite (schéma aligné sur les jeux Kaggle / CrunchBase académique), ou adapter les noms de tables.

**Note G5 :** le bloc G5 réutilise les CTE `top_names`, `vc_investments` et `vc_first_round` définies dans G4 ; exécuter dans un même script ou fusionner les CTE.

---

## G1, Taux d'exit Top VC vs marché

```sql
WITH top_ids AS (
    SELECT id
    FROM objects
    WHERE name IN (
        'Intel Capital',
        'New Enterprise Associates',
        'Sequoia Capital',
        'SV Angel',
        'Accel Partners',
        'Y Combinator',
        'Kleiner Perkins Caufield & Byers',
        'Draper Fisher Jurvetson (DFJ)',
        '500 Startups',
        'First Round Capital'
    )
),
startup_groups AS (
    SELECT
        i.funded_object_id AS object_id,
        CASE
            WHEN MAX(CASE WHEN i.investor_object_id IN (SELECT id FROM top_ids) THEN 1 ELSE 0 END) = 1
            THEN 'Top VC'
            ELSE 'Marche'
        END AS group_label
    FROM investments i
    GROUP BY i.funded_object_id
),
exits_2003_2013 AS (
    SELECT DISTINCT acquired_object_id AS object_id
    FROM acquisitions
    WHERE strftime('%Y', acquired_at) BETWEEN '2003' AND '2013'
)
SELECT
    sg.group_label,
    COUNT(*) AS n_startups_financed,
    SUM(CASE WHEN e.object_id IS NOT NULL THEN 1 ELSE 0 END) AS n_acquired,
    ROUND(100.0 * SUM(CASE WHEN e.object_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 1) AS exit_rate_pct
FROM startup_groups sg
LEFT JOIN exits_2003_2013 e ON e.object_id = sg.object_id
GROUP BY sg.group_label;
```

## G2, Délai médian d'exit, post-traitement pandas pour médiane

```sql
WITH top_ids AS (
    SELECT id
    FROM objects
    WHERE name IN (
        'Intel Capital',
        'New Enterprise Associates',
        'Sequoia Capital',
        'SV Angel',
        'Accel Partners',
        'Y Combinator',
        'Kleiner Perkins Caufield & Byers',
        'Draper Fisher Jurvetson (DFJ)',
        '500 Startups',
        'First Round Capital'
    )
),
group_labels AS (
    SELECT
        i.funded_object_id AS object_id,
        CASE
            WHEN MAX(CASE WHEN i.investor_object_id IN (SELECT id FROM top_ids) THEN 1 ELSE 0 END) = 1
            THEN 'Top VC'
            ELSE 'Marche'
        END AS group_label
    FROM investments i
    GROUP BY i.funded_object_id
),
min_funding AS (
    SELECT object_id AS funded_object_id, MIN(funded_at) AS first_funded_at
    FROM funding_rounds
    WHERE funded_at IS NOT NULL
    GROUP BY object_id
),
raw_delays AS (
    SELECT
        gl.group_label,
        a.acquired_object_id AS object_id,
        strftime('%Y', a.acquired_at) AS exit_year,
        (julianday(a.acquired_at) - julianday(mf.first_funded_at)) / 365.25 AS delay_years
    FROM acquisitions a
    JOIN group_labels gl ON gl.object_id = a.acquired_object_id
    JOIN min_funding mf ON mf.funded_object_id = a.acquired_object_id
    WHERE strftime('%Y', a.acquired_at) BETWEEN '2003' AND '2013'
)
SELECT group_label, object_id, exit_year, delay_years
FROM raw_delays
WHERE delay_years >= 0;
-- Post traitement pandas:
-- q = [0.25, 0.50, 0.75] avec interpolation='nearest'
-- stats = df.groupby('group_label')['delay_years']
--         .quantile(q, interpolation='nearest')
--         .unstack()
```

## G3, Prix d'exit médian + percentiles

```sql
WITH top_ids AS (
    SELECT id
    FROM objects
    WHERE name IN (
        'Intel Capital',
        'New Enterprise Associates',
        'Sequoia Capital',
        'SV Angel',
        'Accel Partners',
        'Y Combinator',
        'Kleiner Perkins Caufield & Byers',
        'Draper Fisher Jurvetson (DFJ)',
        '500 Startups',
        'First Round Capital'
    )
),
group_labels AS (
    SELECT
        i.funded_object_id AS object_id,
        CASE
            WHEN MAX(CASE WHEN i.investor_object_id IN (SELECT id FROM top_ids) THEN 1 ELSE 0 END) = 1
            THEN 'Top VC'
            ELSE 'Marche'
        END AS group_label
    FROM investments i
    GROUP BY i.funded_object_id
),
min_funding AS (
    SELECT object_id AS funded_object_id, MIN(funded_at) AS first_funded_at
    FROM funding_rounds
    WHERE funded_at IS NOT NULL
    GROUP BY object_id
),
eligible AS (
    SELECT
        gl.group_label,
        a.acquired_object_id AS object_id,
        CAST(a.price_amount AS REAL) AS price_usd,
        (julianday(a.acquired_at) - julianday(mf.first_funded_at)) / 365.25 AS delay_years
    FROM acquisitions a
    JOIN group_labels gl ON gl.object_id = a.acquired_object_id
    JOIN min_funding mf ON mf.funded_object_id = a.acquired_object_id
    WHERE strftime('%Y', a.acquired_at) BETWEEN '2003' AND '2013'
)
SELECT group_label, object_id, price_usd
FROM eligible
WHERE delay_years >= 0
  AND price_usd > 0
  AND price_usd < 26000000000;
-- Post traitement pandas:
-- q = [0.50, 0.75, 0.90] avec interpolation='nearest'
-- stats = df.groupby('group_label')['price_usd']
--         .quantile(q, interpolation='nearest')
--         .unstack()
```

## G4, Délai médian par VC, classement individuel

```sql
WITH top_names AS (
    SELECT name
    FROM objects
    WHERE name IN (
        'Intel Capital',
        'New Enterprise Associates',
        'Sequoia Capital',
        'SV Angel',
        'Accel Partners',
        'Y Combinator',
        'Kleiner Perkins Caufield & Byers',
        'Draper Fisher Jurvetson (DFJ)',
        '500 Startups',
        'First Round Capital'
    )
),
vc_investments AS (
    SELECT DISTINCT
        inv.name AS investor_name,
        i.funded_object_id AS object_id,
        i.funding_round_id
    FROM investments i
    JOIN objects inv ON inv.id = i.investor_object_id
    WHERE inv.name IN (SELECT name FROM top_names)
),
vc_first_round AS (
    SELECT
        vi.investor_name,
        vi.object_id,
        MIN(fr.funded_at) AS first_vc_funded_at
    FROM vc_investments vi
    JOIN funding_rounds fr ON fr.funding_round_id = CAST(vi.funding_round_id AS TEXT)
    WHERE fr.funded_at IS NOT NULL
    GROUP BY vi.investor_name, vi.object_id
),
base AS (
    SELECT
        vfr.investor_name,
        vfr.object_id,
        (julianday(a.acquired_at) - julianday(vfr.first_vc_funded_at)) / 365.25 AS delay_years
    FROM vc_first_round vfr
    JOIN acquisitions a ON a.acquired_object_id = vfr.object_id
    WHERE strftime('%Y', a.acquired_at) BETWEEN '2003' AND '2013'
)
SELECT DISTINCT investor_name, object_id, delay_years
FROM base
WHERE delay_years >= 0;
-- Post traitement pandas:
-- q = [0.25, 0.50, 0.75] avec interpolation='nearest'
-- stats = df.groupby('investor_name')['delay_years']
--         .quantile(q, interpolation='nearest')
--         .unstack()
```

## G5, Prix médian par VC, classement individuel

```sql
-- Reutilise les CTE de G4 : top_names, vc_investments, vc_first_round
WITH eligible AS (
    SELECT
        vfr.investor_name,
        vfr.object_id,
        CAST(a.price_amount AS REAL) AS price_usd,
        (julianday(a.acquired_at) - julianday(vfr.first_vc_funded_at)) / 365.25 AS delay_years
    FROM vc_first_round vfr
    JOIN acquisitions a ON a.acquired_object_id = vfr.object_id
    WHERE strftime('%Y', a.acquired_at) BETWEEN '2003' AND '2013'
),
price_ready AS (
    SELECT DISTINCT investor_name, object_id, price_usd
    FROM eligible
    WHERE delay_years >= 0
      AND price_usd > 0
      AND price_usd < 26000000000
)
SELECT investor_name, object_id, price_usd
FROM price_ready;
-- Post traitement pandas:
-- q = [0.50, 0.75, 0.90] avec interpolation='nearest'
-- stats = df.groupby('investor_name')['price_usd']
--         .quantile(q, interpolation='nearest')
--         .unstack()
-- puis flag n<20
```

## G6, Délai médian avant 2008 et après 2008 par groupe

```sql
-- Reutilise exactement les CTE de G2 : top_ids, group_labels, min_funding
WITH base AS (
    SELECT
        gl.group_label,
        CASE
            WHEN CAST(strftime('%Y', a.acquired_at) AS INT) < 2008 THEN 'Pre-2008'
            ELSE 'Post-2008'
        END AS periode,
        a.acquired_object_id AS object_id,
        (julianday(a.acquired_at) - julianday(mf.first_funded_at)) / 365.25 AS delay_years
    FROM acquisitions a
    JOIN group_labels gl ON gl.object_id = a.acquired_object_id
    JOIN min_funding mf ON mf.funded_object_id = a.acquired_object_id
    WHERE strftime('%Y', a.acquired_at) BETWEEN '2003' AND '2013'
)
SELECT group_label, periode, object_id, delay_years
FROM base
WHERE delay_years >= 0;
-- Post traitement pandas:
-- q = [0.25, 0.50, 0.75] avec interpolation='nearest'
-- stats = df.groupby(['group_label','periode'])['delay_years']
--         .quantile(q, interpolation='nearest')
--         .unstack()
```

