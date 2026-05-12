# Occupational Employment Decline Analysis (2024–2034)

A reproducible pipeline that merges four labor-market data sources and uses logistic and linear regression to predict which U.S. occupations are projected to experience employment decline by 2034. The analysis combines BLS Employment Projections, BLS JOLTS labor-market flow rates, the Anthropic Economic Index of AI exposure, and O*NET skill ratings into a single occupation-level dataset, then asks which job characteristics — and which specific skills — are most strongly associated with projected decline and with AI exposure.

## Headline findings

- 29.3% of 832 occupations are projected to decline in employment between 2024 and 2034.
- A logistic regression on 8 features achieves a 5-fold cross-validated AUC of 0.77.
- The skills most associated with projected decline are manual/operational (Equipment Maintenance, Repairing, Operation and Control). The skills most exposed to AI (Programming, Reading Comprehension, Speaking, Writing) are associated with employment growth, not decline.
- Adding O*NET skills to the model does not improve cross-validated prediction; the predictive signal is already captured by the base features. Skills add interpretive depth but not predictive lift.

## Repository structure

```
.
├── README.md
├── requirements.txt
├── data/
│   ├── Employment_Projections.csv           # BLS Employment Projections 2024-2034
│   ├── soc_to_jolts_lookup.csv              # Hand-built SOC major group → NAICS crosswalk
│   ├── jolts_2024_rates.csv                 # BLS JOLTS hires/openings/quits rates
│   ├── aei_exposure_by_soc_major_group.csv  # Anthropic Economic Index, Figure 2
│   ├── aei_top10_occupations.csv            # Anthropic Economic Index, Figure 3
│   └── skills_by_soc_no_commas.csv          # O*NET Skills, reshaped to wide format
├── merge_datasets_v2.ipynb                  # Builds the merged dataset
├── analysis_notebook_v4.ipynb               # Modeling and analysis
└── (generated outputs)
    ├── Employment_Projections_with_JOLTS_AI_and_Skills.csv
    └── Predictions_by_Occupation.csv
```

## Quick start

```bash
# 1. Clone and install dependencies
git clone <this-repo>
cd <this-repo>
pip install -r requirements.txt

# 2. Launch Jupyter
jupyter notebook

# 3. Run the notebooks in order:
#    - merge_datasets_v2.ipynb     produces the merged CSV
#    - analysis_notebook_v4.ipynb  runs the modeling
```

The merge notebook produces `Employment_Projections_with_JOLTS_AI_and_Skills.csv` in the project root. Move it (or copy it) into `data/` before running the analysis notebook, which reads from `data/`.

## Data sources

| Source | What it provides | Grain | Coverage |
|---|---|---|---|
| BLS Employment Projections | 2024 employment, 2034 projection, wages, education | SOC code | 832 occupations |
| BLS JOLTS (2024 annual averages) | Hires, openings, and quits rates | NAICS industry | 17 industries |
| Anthropic Economic Index | Theoretical and observed AI exposure, risk tier | SOC major group + 10 per-occupation overrides | 832 (via fallback) |
| O*NET database (v30.2) | Importance ratings on 35 standardized skills | SOC code | 774 occupations → 751 BLS matches (90.3%) |

## Join architecture

The SOC code is the spine. JOLTS rates join through a hand-built mapping from each occupation's SOC major group (first 2 digits) to its nearest NAICS industry, with three confidence tiers (high/medium/low). AEI joins at the SOC major-group level with per-occupation overrides for the 10 most-exposed occupations. O*NET joins directly on the 6-digit SOC code.

The merge notebook produces an 832 × 54 dataset: 9 BLS columns + 5 JOLTS columns + 5 AEI columns + 35 O*NET skill columns. 81 of the 832 BLS occupations have no O*NET skill data (mostly "all other" catch-all codes that O*NET does not profile separately) — these rows have NaN in the skill columns and are dropped from skill-specific analyses.

## Analytical approach

**Question 1 — likelihood of decline.** Logistic regression predicts the binary target `Will Decline` (1 if projected 2034 employment is below 2024 employment, else 0). Performance evaluated with 5-fold cross-validated AUC. Threshold analysis reports precision, recall, and F1 across seven cut-offs (0.20 to 0.50).

**Question 2 — what characteristics drive decline.** Standardized logistic coefficients identify the strongest predictors. A companion linear regression on the continuous percent-change target confirms direction. Variance inflation factors checked before fitting; the AI Displacement Risk Tier was dropped after this check (it was derived from the AI exposure values, correlation 0.97, VIF above 30).

**Question 3 — skills and decline.** Three analyses:
1. Univariate Pearson correlation between each of the 35 O*NET skills and the binary decline target.
2. Principal Component Analysis on the standardized skill matrix to extract combined-skill factors. The first two components explain 67% of skill variance and have clean interpretations (PC1 = general cognitive skills, PC2 = physical/operational skills).
3. Three-way model comparison on the 751-occupation skills subset: base features only, base + 35 raw skills, base + 4 PCs.

## Reproducing the analysis from scratch

If you want to reconstruct the input files from their original sources rather than using the included `data/` directory:

| File | How to reproduce |
|---|---|
| `Employment_Projections.csv` | Download from https://www.bls.gov/emp/, 2024–2034 release |
| `jolts_2024_rates.csv` | Hand-compiled from BLS JOLTS news release tables 16 (job openings), 18 (hires), and 22 (quits) at https://www.bls.gov/news.release/jolts.toc.htm |
| `aei_exposure_by_soc_major_group.csv`, `aei_top10_occupations.csv` | Hand-compiled from the Anthropic Economic Index paper (Massenkoff & McCrory, March 2026), Figures 2 and 3 |
| `soc_to_jolts_lookup.csv` | Hand-built mapping of the 22 SOC major groups to JOLTS industries |
| `skills_by_soc_no_commas.csv` | Download `Skills.xlsx` from https://www.onetcenter.org/database.html (v30.2), reshape from long to wide format (one row per O*NET-SOC × skill × scale → one row per SOC with 35 skill columns). Reshape code is in `merge_datasets_v2.ipynb` notes |

## Limitations

- The dataset contains projected employment at the occupation level, not unemployment data at the worker level. The "P(Decline)" produced by the model is the probability that an occupation contracts, not the probability that any specific worker becomes unemployed.
- Cross-validated R² for the linear model is ~0.18, meaning about 82% of the variance in projected percent change is unexplained by these features. BLS projections incorporate inputs not in this file (demographic forecasts, industry demand models, capital substitution).
- JOLTS rates are industry averages, not occupation-specific. For cross-industry occupations (management, office support), the mapped rates are weakly informative — flagged in the `Mapping Confidence` column.
- AEI exposure values are at the SOC major-group level for most occupations. Per-occupation AEI data exists in Anthropic's full `job_exposure.csv` release and would tighten this if substituted.
- All numerical results are based on projections through 2034, not measured outcomes.

## Use of AI tools

This project incorporates two distinct uses of AI:

1. **As a data source.** The Anthropic Economic Index (Massenkoff & McCrory, March 2026) provides the AI exposure measures used as predictors. The "observed exposure" measure reflects empirical Claude usage on tasks aligned to O*NET, not expert speculation.
2. **As a development assistant.** Claude (Anthropic) was used throughout the development pipeline — for data cleaning, building the merge and analysis notebooks, diagnosing multicollinearity, and drafting documentation. All numerical results come from standard supervised-learning routines on the merged dataset.

## Sources

- BLS Employment Projections, 2024–2034 release. https://www.bls.gov/emp/
- BLS JOLTS news release tables 16, 18, 22 (March 2026). https://www.bls.gov/news.release/jolts.toc.htm
- Anthropic Economic Index — Massenkoff & McCrory (March 2026). https://www.anthropic.com/research/labor-market-impacts
- O*NET database v30.2, Skills file. https://www.onetcenter.org/database.html
- Eloundou, T., Manning, S., Mishkin, P., & Rock, D. (2023). *GPTs are GPTs.* arXiv:2303.10130.

## License

Code is released under MIT. Underlying data sources retain their own licenses:
- BLS data is public domain.
- O*NET data is CC BY 4.0 (attribute O*NET® / U.S. Department of Labor).
- Anthropic Economic Index data is released under Anthropic's terms.
