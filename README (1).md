# Replicating the Excelsior Scholarship's Enrollment Effects

**A partial replication of Joseph E. Howard's (2026) dissertation, _Excelsior Scholarship and College Enrollment in New York: Evidence of Unexpected Gains_.**

## Project overview

New York introduced the Excelsior Scholarship in 2017 to help eligible residents cover tuition at public colleges and universities. In his dissertation, Howard (2026) examines whether enrollment at New York institutions changed following the program's introduction, using Pennsylvania as a comparison state.

For this project, I recreate the dissertation's main public-institution findings using institution-level data from 2012 through 2019. I first construct a longitudinal dataset from IPEDS and Bureau of Labor Statistics (BLS) data. I then estimate difference-in-differences (DiD) and event-study models to examine changes in enrollment following Excelsior's introduction.

This is a **partial, modified replication**. I focus on two enrollment outcomes and exclude Penn State campuses because their 2019 Pell Grant counts were missing from the IPEDS files I downloaded. This changes the comparison group relative to Howard's original sample.

## Research questions

1. Did first-time, full-time undergraduate enrollment change at New York public institutions following the introduction of the Excelsior Scholarship?
2. Did first-time, full-time enrollment among Pell Grant recipients change over the same period?
3. Do the estimated effects differ when I use a single post-treatment DiD coefficient or year-specific event-study coefficients?

## Data

| Source | Variables used | Coverage |
| --- | --- | --- |
| [IPEDS, National Center for Education Statistics](https://nces.ed.gov/ipeds/use-the-data) | Institution identifiers, enrollment counts, Pell Grant recipient counts, tuition and fees, institution control, and institutional location | Fall 2012–2019 |
| [Bureau of Labor Statistics, Local Area Unemployment Statistics](https://www.bls.gov/lau/) | County unemployment rates | 2011–2018, lagged one year |
| County FIPS crosswalk | County names and five-digit FIPS codes used to merge the BLS data | Matched to institution locations |

**Unit of analysis:** Institution-year.  
**Treatment group:** Public institutions in New York.  
**Comparison group:** Public institutions in Pennsylvania.  
**Study period:** 2012–2019.  
**Policy introduction:** 2017.

The final sample used for the results reported below contains **111 public institutions (74 in New York, 37 in Pennsylvania), observed over eight years**, for **888 institution-year observations**. Howard's original public-institution sample contains 125 institutions (74 in New York, 51 in Pennsylvania).

### Main variables

| Variable in the cleaned panel | Role in the analysis |
| --- | --- |
| `unitid` | Institution identifier; institution fixed effects and clustering |
| `fall_year` | Year identifier; year fixed effects |
| `state` | Treatment (`NY`) and comparison (`PA`) membership |
| `Grand total` | First-time, full-time undergraduate enrollment outcome, as selected from the IPEDS enrollment extract |
| `Number of full-time first-time undergraduates awarded Pell grants` | First-time, full-time Pell Grant enrollment outcome |
| `public` | Indicator for public institutions |
| `Published in-state tuition and fees` | Tuition measure used to construct `ln_tuition` in adjusted DiD models |
| `unemp_lag` | County unemployment rate lagged one year |

The cleaning notebook also brings in race- and gender-disaggregated enrollment counts. I retain these in the broader panel, although they are outside the scope of the main models reported here. The IPEDS `Level of student` field should be checked when recreating the original enrollment extracts to confirm the intended cohort.

## Data cleaning

The cleaning workflow is documented in [`Data_cleaning.ipynb`](Data_cleaning.ipynb).

1. **Merge annual IPEDS data.** For each year, merge the Pell Grant, race/gender enrollment, and tuition files by `unitid`. Resolve duplicate columns and standardize variable names across years.
2. **Create the institution-year panel.** Stack the cleaned annual datasets and add `fall_year`.
3. **Add county identifiers.** Match IPEDS county names to county FIPS codes through a county crosswalk.
4. **Merge unemployment data.** Clean the BLS annual county files, build five-digit FIPS codes, and merge unemployment from the preceding calendar year (`unemp_year = fall_year - 1`).
5. **Identify public institutions.** Use IPEDS institutional control data to restrict the analysis to public institutions.
6. **Apply institutional exclusions.** Remove specialty institutions using the dissertation's exclusion list and exclude all Pennsylvania State University campuses for this modified replication.
7. **Export the cleaned panel.** Save the result as `panel_clean.csv`.

### Penn State data limitation

The 2019 IPEDS extract contains missing Pell Grant recipient counts for Penn State campuses. An earlier version of my cleaning code aggregated Penn State campus counts; summing groups with only missing observations could turn missing values into zeros. I subsequently chose to **exclude all Penn State campuses** to keep a consistent analytic sample without imputing those counts.

**Before reproducing the results below, make sure the Penn State aggregation cell in `Data_cleaning.ipynb` has been replaced with the exclusion used in the final analysis:**

```python
psu = panel["institution name"].str.startswith(
    "Pennsylvania State University",
    na=False
)
panel = panel.loc[~psu].copy()
```

The originally uploaded copy of the cleaning notebook still contained the earlier aggregation code. Updating that cell and rerunning the notebook is necessary for the public GitHub version to match the results documented here.

## Empirical strategy

The analysis is documented in [`Excelsior_Replication_Step_by_Step.ipynb`](Excelsior_Replication_Step_by_Step.ipynb). It follows the dissertation's main public-sector specifications and compares the estimates against Howard's published Tables 10–13.

### 1. Parallel trends

I plot average enrollment by state and year and test whether New York and Pennsylvania had different *linear slopes* in the pretreatment years. The formal test uses an interaction between New York membership and a linear time trend, with institution and year fixed effects and institution-clustered standard errors. I assess pretreatment periods ending in 2016 and 2017 for the two potential treatment start dates.

Failure to reject a difference in pretreatment slopes provides some support for the parallel trends assumption. It does not establish that the assumption is true.

### 2. Difference-in-differences

For each outcome, I estimate a two-way fixed-effects model:

$$
Y_{it} = \alpha + \gamma_i + \lambda_t + \beta(\text{NY}_i \times \text{Post}_t) + \varepsilon_{it}
$$

Here, $Y_{it}$ is enrollment at institution $i$ in year $t$; $\gamma_i$ and $\lambda_t$ are institution and year fixed effects. I estimate separate models defining `Post` as **2017 onward** and **2018 onward**. Adjusted specifications add the natural logarithm of published in-state tuition and fees and lagged county unemployment.

### 3. Event study

I estimate separate New York-by-year coefficients for 2012–2015 and 2017–2019, excluding **2016 as the reference year**:

$$
Y_{it} = \alpha + \gamma_i + \lambda_t
+ \sum_{k \ne 2016}\delta_k(\text{NY}_i \times \mathbf{1}[t=k])
+ \varepsilon_{it}
$$

This model allows the estimated enrollment difference to vary across years. The coefficients represent changes in New York relative to Pennsylvania, measured against the difference in 2016. I also examine 95% confidence intervals and test whether the pretreatment and post-treatment coefficients are jointly equal to zero.

**Inference:** All reported models use institution-clustered standard errors. The event-study results below correspond to the unadjusted fixed-effects specification. The results tables reflect my latest notebook run; the saved analysis notebook may use a different small-sample p-value convention (`use_t=True`), so p-values should be checked when rerunning it.

## Main findings

### Difference-in-differences

Adding tuition and lagged unemployment reduced the estimated treatment coefficients in each specification. All adjusted coefficients remained positive, although none were statistically significant at the 5% level.

| Outcome | Treatment starts | Unadjusted estimate | Adjusted estimate | Adjusted p-value |
| --- | ---: | ---: | ---: | ---: |
| First-time, full-time enrollment | 2017 | 82.99 | 38.26 | 0.444 |
| Pell Grant enrollment | 2017 | 50.64 | 27.14 | 0.319 |
| First-time, full-time enrollment | 2018 | 85.78 | 58.33 | 0.196 |
| Pell Grant enrollment | 2018 | 46.11 | 30.30 | 0.238 |

These results are consistent with the general pattern of Howard's DiD estimates: positive point estimates with substantial uncertainty, particularly after accounting for covariates.

### Event-study estimates

The event studies identify positive post-treatment coefficients for both outcomes. The first-time, full-time enrollment estimates are significant at the 5% level in all three post-treatment years. For Pell Grant recipients, the 2017 and 2018 coefficients are significant at the 5% level; the 2019 coefficient is marginal at the 10% level.

| Year | FTFT: my estimate | FTFT: Howard (2026) | Pell FTFT: my estimate | Pell FTFT: Howard (2026) |
| --- | ---: | ---: | ---: | ---: |
| 2017 | **63.65** (p = 0.004) | 68.35 | **43.97** (p = 0.001) | 51.38 |
| 2018 | **100.01** (p = 0.002) | 83.29 | **56.69** (p = 0.005) | 51.95 |
| 2019 | **102.05** (p = 0.017) | 79.21 | 48.84 (p = 0.058) | 44.90 |

All individual pretreatment event-study coefficients were statistically insignificant at the 5% level. A joint pretreatment-coefficient test is included in the analysis workflow and should also be reported when its output is available.

**Interpretation:** The results suggest increases in enrollment at New York public institutions relative to Pennsylvania following Excelsior's introduction. They generally align with the dissertation's main findings, while differences in sample construction affect the point estimates. These comparisons remain subject to DiD identification assumptions and should not be taken as proof of the scholarship's causal effect.

## Repository contents

The notebooks expect raw inputs in the following locations. The layout below describes the **expected local directory structure**; raw input files and `panel_clean.csv` must be supplied or generated before the analysis can run.

```text
.
├── README.md
├── Data_cleaning.ipynb
├── Excelsior_Replication_Step_by_Step.ipynb
├── IPEDS Data/
│   ├── Pell Grant/
│   │   └── Pell_2012_2013.csv ... Pell_2019_2020.csv
│   ├── Race/
│   │   └── 2012_2013_Race.csv ... 2019_2020_Race.csv
│   └── Tuition and Fees/
│       ├── Tuition_2012-2016.csv
│       └── Tuition_2016-2019.csv
├── BLS Data/
│   └── laucnty11.xlsx ... laucnty18.xlsx
├── Other Data/
│   ├── county_fips_master.csv
│   ├── 2019_county.csv
│   ├── 2012_control.csv
│   └── 2019_control.csv
└── panel_clean.csv                     # generated by data cleaning
```

The analysis notebook can also export `replication_did_comparison.csv`, `replication_event_study_comparison.csv`, and `replication_joint_tests.csv` in the working directory. Exact filenames matter: for example, the current cleaning notebook reads `Pell__2015_2016.csv` with two underscores.

## Reproducing the analysis

1. **Set up Python/Jupyter.** Install the packages used by the notebooks:

   ```bash
   python -m pip install pandas numpy matplotlib statsmodels openpyxl notebook
   ```

2. **Obtain and place the input data.** Download the corresponding IPEDS files and BLS county annual-average files. Keep the input folders and filenames consistent with the paths in `Data_cleaning.ipynb`. The county FIPS crosswalk and institutional-control files are also required.
3. **Run the cleaning notebook.** Confirm that it excludes every Penn State campus, then run the cells in order and generate `panel_clean.csv`.
4. **Run the analysis notebook.** Open `Excelsior_Replication_Step_by_Step.ipynb`. If starting from the saved data, begin with its analysis section, which can load `panel_clean.csv`, instead of rerunning the copied cleaning cells.
5. **Review sample counts and model outputs.** Check that the analysis sample is 74 New York and 37 Pennsylvania institutions per year. Compare the DiD and event-study tables with the reported results above and the dissertation's Tables 10–13.

## Limitations and replication status

- **Modified sample:** Howard includes 51 Pennsylvania public institutions; this project excludes Penn State campuses and retains 37. The resulting estimates are not an exact numerical reproduction of the dissertation.
- **Missing 2019 Penn State data:** The missing Pell Grant values motivate the exclusion and limit direct comparability.
- **Tuition definition:** The adjusted models use the log of published in-state tuition and fees. The exact tuition variable used in Howard's models still needs to be verified.
- **Data construction:** The analysis depends on accurately selecting the IPEDS first-time, full-time undergraduate cohort and correctly aligning financial aid reporting years with fall enrollment years.
- **Identification:** The findings require credible parallel trends and an absence of other New York–Pennsylvania shocks that affect enrollment differently during the study period. Nonsignificant pretrend tests alone cannot verify those assumptions.
- **Scope:** This project reproduces the main public-institution enrollment findings. It does not recreate every institutional sector, outcome, or sensitivity analysis in the dissertation.

## Reference and attribution

Howard, J. E. (2026). *Excelsior scholarship and college enrollment in New York: Evidence of unexpected gains* [Doctoral dissertation, University of Pennsylvania].

The original dissertation is credited for the research design, comparison group, and published benchmark estimates. The data-cleaning and Python replication workflow documented here are part of this independent replication exercise.
