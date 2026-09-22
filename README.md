# Exploring Energy Transition with SQL

*This project aims to demonstrate SQL and general data analysis skills on real data related to the energy industry.*

## Key findings

- Over 2020-2025, renewables' share of global electricity rose from 28.0% to 33.8%, driven mainly by solar and wind, while the fossil share fell from 62.1% to 57.4% - although fossil fuels remain the world's main source of electricity.
- Of the larger generators (over 100TWh), Germany and the UK saw the largest rise in renewables share over 2003-2023 (46.9pp and 43.7pp). In 2023, Norway, Brazil and Sweden had the highest renewables shares, all generating a substantial amount of electricity via hydro.
- Transition speed does not seem to be related to generation size (r = 0.04), and GDP per capita seems at most weakly related to renewables share (r = -0.13 globally, r = -0.17 within regions).

**SQL used:** views, self-joins, CTEs, window functions (`NTILE`, `PERCENT_RANK`), aggregate functions (`CORR`, `STDDEV`, generation-weighted `SUM/SUM`) and an anti-join (`NOT EXISTS`).


## Summary

### Questions at a glance:
1. How has the global electricity mix shifted over the past five years?
2. Which countries have the highest renewable share in 2023?
3. Which countries have grown their renewable share the most over the last 20 years?
4. Which countries have reduced their fossil fuel share the most over the same period?
5. Is transition speed related to the size of a country's electricity generation?
6. How does renewables adoption differ by region, and does relative economic standing matter?

**Q1** looks descriptively at the global electricity mix to set a baseline for the exploration. **Q2** then explores which countries are leading in terms of renewables. **Q3** and **Q4** look at which countries are shifting fastest toward renewables and away from fossil fuels, respectively. **Q5** tests whether larger electricity generators transition more slowly than smaller ones, given the intuitive hypothesis that they may be more locked into existing fossil fuel infrastructure. **Q6** then narrows in on regional patterns, including whether a country's economic standing relative to its regional peers relates to its renewables adoption.


 ### Datasets
- Our World in Data (OWID)'s "Energy" dataset
- World Bank's country-region classification (2025)

The two datasets are joined via ISO code into a single view, `focus_countries`, used throughout Q2-Q6:

```sql
CREATE VIEW focus_countries AS
SELECT *
FROM owid_energy_data
INNER JOIN wb_data USING (iso_code)
```

### Scope at a glance
- Countries in `focus_countries`: 207 (199 with renewables share data in both 2003 and 2023)
- 7 World Bank regions (see Q6 for a breakdown by country count and electricity generation)
- Data from 2020-2025 (Q1), 2003 and 2023 (Q2-Q5), and 2022 (Q6)
- Electricity generation specifically, not energy generation overall

### Tools
- VS Code
- PostgreSQL

### Methodological decisions
- **Electricity rather than total energy**: electricity generation was chosen over broader energy metrics, as it's more directly relevant to an energy-sector audience and captures the transition most visibly, while total energy also includes transport, heating, and industrial fuel use.
- **World Bank regions over OWID's own groupings**: World Bank's 7 regions were used instead of OWID's continent-based classification, as they group countries in ways more reflective of shared economic and geographic context (e.g. separating North America from Latin America & the Caribbean).
- **2003-2023 as the comparison window**: a 20-year baseline ending in 2023.
- **100TWh threshold (Q2-4)**: leaderboard-style questions are restricted to countries generating over 100TWh, to focus on larger, more established generators.

 
### Limitations
- The 100TWh threshold used in Q2-Q4 was chosen as a round number to focus on larger, established generators, not derived from the data distribution - a more rigorous approach (e.g. percentile-based, or informed by inspecting the generation distribution for a natural cutoff) would strengthen this choice. It also reduces the sample considerably, from 199 countries to 25 (or 36, for Q2's single-year filter). See Further Work.
- 13 countries in the OWID dataset have no World Bank match and are excluded from `focus_countries` (Q2-Q6). See Appendix for the full list and query.



## Key Questions

### 1. How has the global electricity mix shifted over the past five years?


**Result snapshot 1:**

|Year|Fossil           |Nuclear           |Renewables           |
|----|-----------------|------------------|---------------------|
|2020|62.1             |9.9               |28.0                 |
|2021|62.1             |9.8               |28.1                 |
|2022|61.4             |9.1               |29.5                 |
|2023|60.6             |9.1               |30.3                 |
|2024|59.1             |9.0               |31.9                 |
|2025|57.4             |8.8               |33.8                 |

*Share of electricity generated for each source (group level); values rounded to 1 decimal place*

**Takeaway:**
Over 2020-2025, the global electricity mix shifted towards renewables and away from nuclear and fossil sources - although fossil fuels remain the world's main source for electricity.

**Query**
```sql
SELECT
    year,
    ROUND(fossil_share_elec::numeric,1) AS fossil_share_elec,
    ROUND(nuclear_share_elec::numeric,1) AS nuclear_share_elec,
    ROUND(renewables_share_elec::numeric,1) AS renewables_share_elec
FROM owid_energy_data
WHERE owid_country = 'World' AND year BETWEEN 2020 AND 2025
ORDER BY year 
```


**Result snapshot 2:**

|Year|Coal           |Gas           |Oil           |Nuclear           |Hydro           |Solar           |Wind           |Other Renewables           |
|----|---------------|--------------|--------------|------------------|----------------|----------------|---------------|---------------------------|
|2020|35.3           |23.9          |2.9           |9.9               |16.3            |3.2             |6.0            |2.6                        |
|2021|35.9           |23.2          |3.0           |9.8               |15.2            |3.7             |6.6            |2.6                        |
|2022|35.4           |22.9          |3.1           |9.1               |15.0            |4.6             |7.3            |2.6                        |
|2023|35.0           |22.6          |3.0           |9.1               |14.3            |5.6             |7.8            |2.6                        |
|2024|34.1           |22.3          |2.8           |9.0               |14.3            |6.9             |8.1            |2.6                        |
|2025|33.0           |21.8          |2.6           |8.8               |14.0            |8.7             |8.5            |2.5                        |


*Granular breakdown of the above, by individual source; values rounded to 1 decimal place*

**Takeaway:**
Solar and wind saw the largest pp gains in share, with increases of 5.5pp and 2.5pp respectively. In terms of output, based on absolute TWh figures checked separately, solar more than tripled (853 to 2,778 TWh) and wind increased by about 70% (1,591 to 2,713 TWh). Hydro's output stayed relatively stable over the period (fluctuating year to year with a small net increase), so its slight share decline reflects being outpaced by faster-growing sources rather than any real drop in its own output. The same is true for nuclear, coal, gas, and oil, which all grew in absolute terms but far more slowly in percentage terms than solar and wind (so their shares still declined despite rising output).


**Query**
```sql
SELECT
    year,
    ROUND(coal_share_elec::numeric,1) AS coal_share_elec,
    ROUND(gas_share_elec::numeric,1) AS gas_share_elec,
    ROUND(oil_share_elec::numeric,1) AS oil_share_elec,
    ROUND(nuclear_share_elec::numeric,1) AS nuclear_share_elec,
    ROUND(hydro_share_elec::numeric,1) AS hydro_share_elec,
    ROUND(solar_share_elec::numeric,1) AS solar_share_elec,
    ROUND(wind_share_elec::numeric,1) AS wind_share_elec,
    ROUND(other_renewables_share_elec::numeric,1) AS other_renewables_share_elec
FROM owid_energy_data
WHERE owid_country = 'World' AND year BETWEEN 2020 AND 2025
ORDER BY year 
```

**Footnote:**
This question uses OWID's global "World" aggregate directly rather than `focus_countries`, so it includes all countries regardless of World Bank match. Absolute TWh figures quoted in the takeaway were checked separately.

<details>
<summary>Data check: do the source shares sum to 100%?</summary>


|Year|Total Share|
|----|-----------|
|2020|100.00     |
|2021|100.00     |
|2022|100.00     |
|2023|100.00     |
|2024|100.00     |
|2025|100.00     |


```sql
SELECT
    year,
    ROUND((coal_share_elec + gas_share_elec + oil_share_elec + nuclear_share_elec
         + hydro_share_elec + solar_share_elec + wind_share_elec
         + other_renewables_share_elec)::numeric, 2) AS total_share
FROM owid_energy_data
WHERE owid_country = 'World' AND year BETWEEN 2020 AND 2025
ORDER BY year
```

</details>

### 2. Which countries have the highest renewable share in 2023?

**Result snapshot:**
|Country  |Electricity Generation (TWh)|Renewables Share|
|--------------|----------------------|---------------------|
|Norway        |153.6                 |98.5                 |
|Brazil        |708.1                 |89.0                 |
|Sweden        |166.1                 |69.4                 |
|Canada        |641.4                 |65.4                 |
|Germany       |506.7                 |54.7                 |
|Spain         |279.1                 |51.5                 |
|Netherlands   |120.0                 |47.4                 |
|United Kingdom|292.5                 |46.4                 |
|Italy         |261.5                 |44.5                 |
|Vietnam       |276.6                 |42.9                 |

*Total electricity generation and renewables share for countries with the highest renewables share and electricity generation above 100TWh; values rounded to 1 decimal place*


**Takeaway:**
Of the larger electricity generators, Norway, Brazil and Sweden were the top 3 in terms of renewable shares. Looking at these countries' electricity mix, all 3 countries generate a substantial amount of their electricity via hydro sources, with Sweden also generating a significant amount in nuclear and wind.

**Query**
```sql
SELECT
    owid_country,
    ROUND(electricity_generation::numeric,1) AS electricity_generation,
    ROUND(renewables_share_elec::numeric,1) AS renewables_share_elec
FROM focus_countries
WHERE year = 2023 AND electricity_generation >100
ORDER BY renewables_share_elec DESC
LIMIT 10
```
   
**Footnote:**
Looking at countries with electricity generation greater than 100TWh. Without this condition, the top 10 would mainly consist of very small countries with 100% renewables shares.


### 3. Which countries have grown their renewable share the most over the last 20 years?

**Result snapshot:**
|Country  |Renewables Share 2003|Renewables Share 2023|pp Change in Share|
|--------------|--------------|--------------|---------------|
|Germany       |7.8           |54.7          |46.9           |
|United Kingdom|2.7           |46.4          |43.7           |
|Spain         |21.7          |51.5          |29.8           |
|Italy         |16.4          |44.5          |28.2           |
|Australia     |8.3           |34.8          |26.6           |
|Poland        |1.5           |27.7          |26.2           |
|Sweden        |43.4          |69.4          |26.0           |
|Turkey        |25.5          |42.0          |16.5           |
|France        |11.0          |26.9          |15.9           |
|China         |15.0          |30.6          |15.6           |

*Top 10 countries in terms of pp change in share from 2003 to 2023, only looking at the large generators*

**Takeaway:**
Of the larger generators, Germany and the UK have seen the largest pp change in renewables share with changes of 46.9pp and 43.7pp respectively, compared to a 14.5pp average among the same group. For both countries, based on absolute TWh figures checked separately, this was driven by a mix of rising solar and wind output alongside declining coal, gas, and oil output - although total electricity generation also fell over 2003-2023, so some of the fossil decline reflects a shrinking grid rather than being fully offset by renewables growth.

**Query 1: Top 10 in terms of pp change in renewable shares among large generators**
```sql
SELECT
    _23.owid_country,
    ROUND(_03.renewables_share_elec::numeric,1) AS renewshare2003,
    ROUND(_23.renewables_share_elec::numeric,1) AS renewshare2023,
    ROUND((_23.renewables_share_elec - _03.renewables_share_elec)::numeric,1) AS change_in_share
FROM focus_countries AS _23 JOIN focus_countries AS _03 USING (iso_code)
WHERE 
    _23.year = 2023 AND _03.year = 2003 
    AND _23.renewables_share_elec IS NOT NULL
    AND _03.renewables_share_elec IS NOT NULL
    AND _23.electricity_generation > 100 
    AND _03.electricity_generation > 100
ORDER BY change_in_share DESC
LIMIT 10
```


**Query 2: Finding (unweighted) average change in share for large generators**
```sql
SELECT
    AVG(_23.renewables_share_elec - _03.renewables_share_elec) AS avg_change_in_share
FROM focus_countries AS _23 JOIN focus_countries AS _03 USING (iso_code)
WHERE 
    _23.year = 2023 AND _03.year = 2003 
    AND _23.renewables_share_elec IS NOT NULL
    AND _03.renewables_share_elec IS NOT NULL
    AND _23.electricity_generation > 100 
    AND _03.electricity_generation > 100
```


**Footnote:** Filtering on electricity generation in both 2003 and 2023 (minimum threshold of 100 TWh in both years) narrows countries down from 207 to 199 (non-null renewables share in both years) to 25 (over 100 TWh in both years).


### 4. Which countries have reduced their fossil fuel share the most over the same period?

**Result snapshot:**
|Country |Fossil Share 2003|Fossil Share 2023|pp Change in Share|
|--------------|---------------|---------------|---------------|
|United Kingdom|75.0           |39.7           |-35.3          |
|Italy         |83.6           |55.5           |-28.2          |
|Australia     |91.7           |65.2           |-26.6          |
|Poland        |98.5           |72.3           |-26.2          |
|Spain         |54.3           |28.1           |-26.2          |
|Germany       |64.8           |43.9           |-20.9          |
|China         |82.7           |64.8           |-17.9          |
|Turkey        |74.5           |58.0           |-16.5          |
|United States |71.2           |59.1           |-12.1          |
|South Africa  |94.1           |84.1           |-9.9           |



**Takeaway:**
Of the larger generators, the UK has the largest pp fall in fossil share with a reduction of 35.3pp, followed by Italy, Australia, Poland and Spain. With the exception of Spain, the top 5 is represented by countries with large fossil fuel shares to begin with. There is substantial overlap between the countries transitioning fastest towards renewables (Q3) and away from fossil fuels (Q4) - Germany, UK, Italy, Spain, Australia and Poland appear in both top lists, suggesting these are the same underlying transitions viewed from two angles (although there is a large discrepancy for Germany, suggesting a large part is also a shift from nuclear generation).


**Query**
```sql
SELECT
    _23.owid_country,
    ROUND(_03.fossil_share_elec::numeric,1) AS fossilshare2003,
    ROUND(_23.fossil_share_elec::numeric,1) AS fossilshare2023,
    ROUND((_23.fossil_share_elec - _03.fossil_share_elec)::numeric,1) AS change_in_share
FROM focus_countries AS _23 JOIN focus_countries AS _03 USING (iso_code)
WHERE 
    _23.year = 2023 AND _03.year = 2003 
    AND _23.fossil_share_elec IS NOT NULL
    AND _03.fossil_share_elec IS NOT NULL
-- looking at countries with already large capacities: threshold is 100twh
    AND _23.electricity_generation > 100 
    AND _03.electricity_generation > 100
ORDER BY change_in_share ASC
LIMIT 10
```

**Footnote:** Filtering on electricity generation in both 2003 and 2023 (minimum threshold of 100 TWh in both years) narrows countries down from 207 to 199 (non-null fossil share in both years) to 25 (over 100 TWh in both years).

### 5. Is transition speed related to the size of a country's electricity generation?

**Hypothesis:** Larger electricity generators (bigger economies/grids) are more "locked in" to existing fossil fuel infrastructure and would be expected to show slower transition speeds (smaller pp change in renewables share) than smaller generators, which may have more flexibility to shift energy mix quickly.

**Result snapshot:**

Headline correlation between electricity generation in 2023 and pp change in renewable share from 2003 to 2023: Pearson correlation r = 0.04, Spearman correlation ρ = 0.09

Looking at generation quartiles

|Generation Quartile|Average Change in Renewable Share|Minimum Share Change|Maximum Share Change|Standard Deviation Change|Minimum Electricity Generation (TWh)|Maximum Electricity Generation (TWh)|Average Electricity Generation (TWh)|
|-------------------|-----------------------------|----------------|----------------|-------------|--------|--------|-------|
|1 (Largest)        |10.3                         |-27.3           |46.9            |15.0         |57.3    |9456.5  |555.9  |
|2                  |8.8                          |-27.1           |68.8            |17.8         |12.8    |57.0    |28.9   |
|3                  |5.1                          |-69.5           |84.3            |25.2         |1.1     |12.8    |5.7    |
|4 (Smallest)       |8.5                          |-39.9           |81.9            |21.3         |0.0     |1.1     |0.5    |

*Quartile 1 reflects the largest generators, 4 is the lowest; changes are considered in pp terms; electricity generation stats given for context*


**Takeaway:**
The correlation between generation size and pp change is negligible (Pearson correlation r = 0.04, Spearman correlation ρ = 0.09). Quartile averages show no consistent directional pattern with size, and every quartile has wide variance (standard deviation 15-25pp) with overlapping min/max ranges.


**Query 1: Pearson Correlation**
```sql
SELECT
    CORR(_23.electricity_generation,_23.renewables_share_elec - _03.renewables_share_elec)
FROM focus_countries AS _23 
JOIN focus_countries AS _03 USING (iso_code)
WHERE 
    _23.year = 2023 AND _03.year = 2003 
    AND _23.renewables_share_elec IS NOT NULL
    AND _03.renewables_share_elec IS NOT NULL
```

**Query 2: Spearman correlation**
```sql
WITH changes AS (
    SELECT
        _23.electricity_generation AS elec2023,
        _23.renewables_share_elec - _03.renewables_share_elec AS change_in_share
    FROM focus_countries AS _23
    JOIN focus_countries AS _03 USING (iso_code)
    WHERE
        _23.year = 2023 AND _03.year = 2003
        AND _23.renewables_share_elec IS NOT NULL
        AND _03.renewables_share_elec IS NOT NULL
        AND _23.electricity_generation IS NOT NULL
),
ranked AS (
    SELECT
        RANK() OVER (ORDER BY elec2023) AS gen_rank,
        RANK() OVER (ORDER BY change_in_share) AS change_rank
    FROM changes
)
SELECT COUNT(*), CORR(gen_rank, change_rank) AS spearman_r
FROM ranked
```

**Query 3: Quartile inspection**
```sql
WITH quartiled AS (
    SELECT
        _23.electricity_generation AS elec2023,
        _23.renewables_share_elec - _03.renewables_share_elec AS change_in_share,
        NTILE(4) OVER (ORDER BY _23.electricity_generation DESC) AS generation_quartile
    FROM focus_countries AS _23 
    JOIN focus_countries AS _03 USING (iso_code)
    WHERE 
        _23.year = 2023 AND _03.year = 2003 
        AND _23.renewables_share_elec IS NOT NULL
        AND _03.renewables_share_elec IS NOT NULL
        AND _23.electricity_generation IS NOT NULL
)
SELECT
    generation_quartile,
    ROUND(MIN(change_in_share)::numeric,1) AS min_share_change,
    ROUND(MAX(change_in_share)::numeric,1) AS max_share_change,
    ROUND(STDDEV(change_in_share)::numeric,1) AS stddev_change,
    ROUND(MIN(elec2023)::numeric,1) AS min_elec,
    ROUND(MAX(elec2023)::numeric,1) AS max_elec,
    ROUND(AVG(elec2023)::numeric,1) AS avg_gen,
    ROUND(AVG(change_in_share)::numeric,1) AS avg_change_in_renewable_share
FROM quartiled
GROUP BY generation_quartile
ORDER BY generation_quartile

```
**Footnote:** Sample size 199.

### 6. How does renewables adoption differ by region, and does relative economic standing matter?

|Region                                           |Country Count|Total Electricity Generation (TWh)|
|-------------------------------------------------|------------|----------------------------|
|East Asia & Pacific                              |36          |12421                       |
|Europe & Central Asia                            |51          |5346                        |
|North America                                    |3           |4952                        |
|Middle East, North Africa, Afghanistan & Pakistan|23          |1998                        |
|South Asia                                       |6           |1946                        |
|Latin America & Caribbean                        |40          |1727                        |
|Sub-Saharan Africa                               |48          |514                         |

*Total electricity generation for each region in 2022, in descending order by total electricity generation*

**Result snapshot:**

|Region                                           |Renewables Share|
|-------------------------------------------------|-----------------------|
|Latin America & Caribbean                        |62.1                   |
|Sub-Saharan Africa                               |36.0                   |
|Europe & Central Asia                            |34.9                   |
|North America                                    |28.5                   |
|East Asia & Pacific                              |27.8                   |
|South Asia                                       |20.3                   |
|Middle East, North Africa, Afghanistan & Pakistan|6.8                    |

*Renewables share in 2022 for each region, as classified by the World Bank, generation-weighted*

World correlation of GDP per capita against renewable share is r = -0.13 (2dp) for 165 countries with GDP, population and renewables share data.

Correlation of intraregion GDP per capita percentile rank against intraregion renewables share percentile rank was ρ = -0.17 (2dp) across 165 countries, a weak negative relationship.

**Takeaway:**

Renewables shares vary widely by region, from 62.1% in Latin America & Caribbean to 6.8% in the Middle East, North Africa, Afghanistan & Pakistan. Shares are generation-weighted, so large generators dominate each region's figure; Latin America's is plausibly driven by hydro-heavy Brazil (Q2), though this wasn't tested directly. On economic standing, GDP per capita is weakly negatively related to renewables share within regions (r = -0.17, using percentile ranks so region size doesn't distort the result). Richer countries are, if anything, slightly less renewables-heavy, but the relationship is weak and GDP per capita explains little of the variation.

**Query 1: Region overview with counts and total electricity generation** 
```sql
SELECT 
    region,
    count(DISTINCT owid_country) AS region_count,
    ROUND(SUM(electricity_generation)::numeric,0) AS total_electricity_generation
FROM focus_countries
WHERE year = 2022
GROUP BY region
ORDER BY SUM(electricity_generation) DESC
```

**Query 2: Weighted renewables share per region**
```sql
SELECT
    region,
    ROUND(((SUM(renewables_electricity)/SUM(electricity_generation))*100)::numeric,1) AS region_renewables_share
FROM focus_countries
WHERE year = 2022
GROUP BY region
ORDER BY region_renewables_share DESC
```

**Query 3: Correlation between renewables share and GDP per capita**

```sql
SELECT
    COUNT(*),
    CORR(renewables_share_elec, (gdp/population))
FROM focus_countries
WHERE year = 2022
AND renewables_share_elec IS NOT NULL
AND gdp IS NOT NULL
AND population IS NOT NULL
```

**Query 4: Correlation between intraregional renewables percentile rank and GDP per capita percentile rank**

```sql
WITH ranked_within_region AS(
    SELECT
        PERCENT_RANK() OVER (PARTITION BY region ORDER BY renewables_share_elec DESC) AS region_renewables_rank,
        PERCENT_RANK() OVER (PARTITION BY region ORDER BY gdp/population DESC) AS region_gdp_pc_rank
    FROM focus_countries
    WHERE year = 2022 
    AND renewables_share_elec IS NOT NULL
    AND gdp IS NOT NULL
    AND population IS NOT NULL
)
SELECT
    COUNT(*),
    CORR(region_renewables_rank, region_gdp_pc_rank)
FROM ranked_within_region
```

**Footnote:** Using 2022 as there is no GDP data for 2023. Region overview table covers all 207 countries with a 2022 row; the correlations use the 165 with GDP, population and renewables share data.


### Conclusions

Overall, there seems to be a shift towards renewables and away from other sources, particularly towards solar and wind; there is a large overlap between renewables growth and fossil decline. While most growth in renewables seems to come from solar and wind, the generators with the highest renewables shares (Norway, Brazil and Sweden) are countries which generate a substantial amount via hydro sources. This is plausibly linked to hydro's storage/dispatchability advantages over intermittent renewables meaning it can be relied on more, though this project doesn't test that directly. 

Speed of transition towards renewables does not appear to be related to generation size (r = 0.04), although this is a limited test: generation size is only a rough proxy for fossil fuel lock-in, and a country's pp change is bounded by its starting share. GDP per capita also shows no clear relationship with renewables share globally (r = -0.13), and only a weak negative one within regions (r = -0.17, using percentile ranks): richer countries tend to be slightly less renewables-heavy than their poorer regional peers, but GDP per capita explains little of the variation.

### Further Work
- Revisit the 100TWh threshold with a percentile-based or distribution-informed cutoff.
- Identify which countries are driving the global increase in renewable generation in absolute terms, rather than which countries show the fastest share growth (Q3), since a country can lead on share growth without contributing much to the global total.

---

### Appendix

**Excluded countries (no WB match):**
- Saint Helena
- Martinique
- Reunion
- Montserrat
- Antarctica
- Cook Islands
- French Guiana
- Netherlands Antilles
- Saint Pierre and Miquelon
- Falkland Islands
- Niue
- Guadeloupe
- Western Sahara

**Query** 
```sql
SELECT
    DISTINCT owid_country
FROM owid_energy_data AS owid
WHERE NOT EXISTS (
    SELECT 1 FROM wb_data AS wb WHERE wb.iso_code = owid.iso_code
)
AND iso_code IS NOT NULL
```




