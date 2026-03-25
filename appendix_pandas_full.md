# Annexe Pandas complète (ADD#2)

Ce fichier reprend **les mêmes extraits** que la section « Requêtes pandas clés » de l’annexe PDF et que la liste `pandas_blocks` dans [`generate_pdf.py`](generate_pdf.py).

Ces blocs illustrent le chaînage après extraction SQL (médianes, quantiles avec `interpolation='nearest'`, déduplication, agrégations par groupe ou par VC).

---

## P1, Groupes Top VC vs marché

```python
top_ids = pd.read_sql_query(
    "SELECT id FROM objects WHERE name IN ({})".format(','.join(['?']*len(top_vc_list))),
    conn, params=top_vc_list
)
top_id_set = set(top_ids['id'].tolist())

investments = pd.read_sql_query(
    "SELECT funded_object_id, investor_object_id FROM investments", conn
)
investments['is_top'] = investments['investor_object_id'].isin(top_id_set).astype(int)
group_labels = (
    investments.groupby('funded_object_id', as_index=False)['is_top']
    .max()
    .rename(columns={'funded_object_id': 'object_id'})
)
group_labels['group_label'] = group_labels['is_top'].map({1: 'Top VC', 0: 'Marche'})
```

## P2, Base délai commune

```python
acq = pd.read_sql_query(
    "SELECT acquired_object_id, acquired_at, price_amount FROM acquisitions", conn
).rename(columns={'acquired_object_id': 'object_id'})
acq['acquired_at_dt'] = pd.to_datetime(acq['acquired_at'], errors='coerce')
acq = acq[acq['acquired_at_dt'].dt.year.between(2003, 2013, inclusive='both')].copy()
acq['price_usd'] = pd.to_numeric(acq['price_amount'], errors='coerce')

min_funding = pd.read_sql_query(
    """
SELECT object_id, MIN(funded_at) AS first_funded_at
FROM funding_rounds
WHERE funded_at IS NOT NULL
GROUP BY object_id
""", conn
)
min_funding['first_funded_at_dt'] = pd.to_datetime(min_funding['first_funded_at'], errors='coerce')

base_delay = acq.merge(group_labels, on='object_id', how='inner').merge(
    min_funding[['object_id', 'first_funded_at_dt']], on='object_id', how='inner'
)
base_delay['delay_years'] = (base_delay['acquired_at_dt'] - base_delay['first_funded_at_dt']).dt.days / 365.25
base_delay = base_delay[pd.to_numeric(base_delay['delay_years'], errors='coerce') >= 0].copy()
```

## P3, Métriques groupe (I1, I2, I3, I6)

```python
def q_nearest(series, q):
    clean = pd.to_numeric(series, errors='coerce').dropna()
    return float(clean.quantile(q, interpolation='nearest')) if not clean.empty else 0.0

g2 = base_delay.groupby('group_label', as_index=False).agg(
    n=('delay_years', 'count'),
    median_years=('delay_years', lambda s: q_nearest(s, 0.50)),
    p25_years=('delay_years', lambda s: q_nearest(s, 0.25)),
    p75_years=('delay_years', lambda s: q_nearest(s, 0.75))
)

g3_base = base_delay[(base_delay['price_usd'] > 0) & (base_delay['price_usd'] < 26_000_000_000)].copy()
g3 = g3_base.groupby('group_label', as_index=False).agg(
    n_with_price=('price_usd', 'count'),
    median_usd=('price_usd', lambda s: q_nearest(s, 0.50)),
    p75_usd=('price_usd', lambda s: q_nearest(s, 0.75)),
    p90_usd=('price_usd', lambda s: q_nearest(s, 0.90))
)

g6_base = base_delay.copy()
g6_base['period'] = g6_base['acquired_at_dt'].dt.year.apply(lambda y: 'pre_2008' if y < 2008 else 'post_2008')
g6 = g6_base.groupby(['group_label', 'period'], as_index=False).agg(
    n=('delay_years', 'count'),
    median_delay_years=('delay_years', lambda s: q_nearest(s, 0.50)),
    p25=('delay_years', lambda s: q_nearest(s, 0.25)),
    p75=('delay_years', lambda s: q_nearest(s, 0.75))
)
```

## P4, Métriques VC individuelles (I4, I5)

```python
vc_investments = pd.read_sql_query(
    """
SELECT DISTINCT inv.name AS investor_name, i.funded_object_id AS object_id, i.funding_round_id
FROM investments i
JOIN objects inv ON inv.id = i.investor_object_id
WHERE inv.name IN ({})
""".format(','.join(['?']*len(top_vc_list))),
    conn, params=top_vc_list
)
vc_investments['funding_round_id_str'] = vc_investments['funding_round_id'].astype(str)

funding_rounds = pd.read_sql_query(
    "SELECT funding_round_id, funded_at FROM funding_rounds WHERE funded_at IS NOT NULL", conn
)
funding_rounds['funding_round_id_str'] = funding_rounds['funding_round_id'].astype(str)
funding_rounds['funded_at_dt'] = pd.to_datetime(funding_rounds['funded_at'], errors='coerce')

vc_rounds = vc_investments.merge(funding_rounds[['funding_round_id_str', 'funded_at_dt']], on='funding_round_id_str', how='inner')
vc_first_round = vc_rounds.groupby(['investor_name', 'object_id'], as_index=False)['funded_at_dt'].min().rename(columns={'funded_at_dt': 'first_vc_funded_at_dt'})
vc_base = vc_first_round.merge(acq[['object_id', 'acquired_at_dt', 'price_usd']], on='object_id', how='inner')
vc_base['delay_years'] = (vc_base['acquired_at_dt'] - vc_base['first_vc_funded_at_dt']).dt.days / 365.25
vc_base = vc_base[pd.to_numeric(vc_base['delay_years'], errors='coerce') >= 0].copy()

g4 = vc_base.drop_duplicates(subset=['investor_name', 'object_id', 'delay_years']).groupby('investor_name', as_index=False).agg(
    n_exits=('delay_years', 'count'),
    median_delay_years=('delay_years', lambda s: q_nearest(s, 0.50)),
    p25=('delay_years', lambda s: q_nearest(s, 0.25)),
    p75=('delay_years', lambda s: q_nearest(s, 0.75))
).sort_values('median_delay_years', ascending=True)

g5_base = vc_base[(vc_base['price_usd'] > 0) & (vc_base['price_usd'] < 26_000_000_000)][['investor_name', 'object_id', 'price_usd']].drop_duplicates()
g5 = g5_base.groupby('investor_name', as_index=False).agg(
    n_exits_with_price=('price_usd', 'count'),
    median_price_usd=('price_usd', lambda s: q_nearest(s, 0.50)),
    p75_usd=('price_usd', lambda s: q_nearest(s, 0.75)),
    p90_usd=('price_usd', lambda s: q_nearest(s, 0.90))
).sort_values('median_price_usd', ascending=False)
```

## P5, Data coverage

```python
n_observations_delay_analysis = int(len(base_delay))
n_top_vc_exits = int(len(base_delay[base_delay['group_label'] == 'Top VC']))
n_market_exits = int(len(base_delay[base_delay['group_label'] == 'Marche']))
n_with_price_top_vc = int(len(g3_base[g3_base['group_label'] == 'Top VC']))
n_with_price_market = int(len(g3_base[g3_base['group_label'] == 'Marche']))
price_missing_rate_pct = round(
    100.0 * (1 - (n_with_price_top_vc + n_with_price_market) / max(1, n_observations_delay_analysis)),
    2
)
```

