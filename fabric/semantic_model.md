# Semantic model: `sm_platsbanken_jobs`

Power BI semantic model on top of the gold lakehouse (`lh_gold`), in Direct Lake mode.
Git sync is disabled in the school's Fabric tenant, so the model is documented here by hand.

## Tables

| Table | Grain | Rows |
|---|---|---|
| `fact_ad` | one row per ad | 11,718 |
| `fact_ad_skill` | one row per ad and skill (bridge) | ~46,000 |
| `dim_date` | one row per day, 2016–2026 (marked as date table) | 4,018 |
| `dim_role` | role | 5 |
| `dim_location` | region × municipality | 158 |
| `dim_employer` | employer name × organisation number | 2,038 |
| `dim_skill` | skill and category | 46 |
| `_Measures` | empty table holding all measures | – |

## Relationships

| From (many) | To (one) | Cross-filter |
|---|---|---|
| `fact_ad[date_key]` | `dim_date[date_key]` | Single |
| `fact_ad[role_key]` | `dim_role[role_key]` | Single |
| `fact_ad[location_key]` | `dim_location[location_key]` | Single |
| `fact_ad[employer_key]` | `dim_employer[employer_key]` | Single |
| `fact_ad_skill[skill_key]` | `dim_skill[skill_key]` | Single |
| `fact_ad_skill[ad_id]` | `fact_ad[ad_id]` | **Both** |

The relationship between the bridge and `fact_ad` is bidirectional. `fact_ad` is the only table that connects a skill to role, date and location, so a filter on a skill has to be able to reach the ads, and a filter on a role has to be able to reach the skills.

Direct Lake does not validate cardinality or cross-filter direction, so these were set by hand.

## Measures

```dax
Ads = COUNTROWS(fact_ad)

Vacancies = SUM(fact_ad[n_vacancies])

Ads with skill = COUNTROWS(fact_ad_skill)

Employers = DISTINCTCOUNT(fact_ad[employer_key])

Median days open = MEDIANX(fact_ad, fact_ad[days_open])

Skill share =
VAR Denominator =
    CALCULATE(
        [Ads],
        REMOVEFILTERS(dim_skill),
        REMOVEFILTERS(fact_ad_skill),
        VALUES(dim_role[role]),
        VALUES(dim_date[year])
    )
RETURN
    DIVIDE([Ads with skill], Denominator)
```

### Why `Skill share` looks the way it does

The measure answers "what share of this role's ads in this period mention this skill". The numerator is easy; the denominator took three attempts.

1. **`DIVIDE([Ads with skill], [Ads])` returned 100% in every cell.** Through the bidirectional relationship, the skill filter on the row also filtered `fact_ad`, so the denominator only counted ads that had the skill — numerator and denominator were the same rows.
2. **Removing the skill filter (`REMOVEFILTERS(dim_skill), REMOVEFILTERS(fact_ad_skill)`) went too far.** The role on the column reaches `fact_ad` partly via the bridge, so removing the bridge filter also dropped the role, and every cell was divided by all 11,718 ads (Python for Data Engineer showed 17.9% instead of 47.4%).
3. **The final version removes the skill filter and then puts role and year back** with `VALUES(...)`. The results match the shares computed independently in Spark in `nb_03_silver_skills` (e.g. Python 47.4% of Data Engineer ads, machine learning 65.8% of Data Scientist ads), which is how the measure was validated.

The same reasoning is why shares are never computed against all ads: the role mix changed a lot over time (BI Developer was 33% of ads in 2019 and 14% in 2025), so a share of all ads mixes "how common is this skill" with "how common is this role".
