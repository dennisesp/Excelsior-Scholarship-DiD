# Replicating the Excelsior Scholarship's Enrollment Effects

*A partial replication of Joseph E. Howard's (2026) dissertation, *Excelsior Scholarship and College Enrollment in New York: Evidence of Unexpected Gains*.*

## Project Overview

New York introduced the Excelsior Scholarship in 2017 to help eligible residents cover tuition at public colleges and universities. In his dissertation, Howard (2026) examines whether enrollment changed following the program's introduction, using Pennsylvania as a comparison state.

For this project, I recreate Howard's main findings for public institutions using data from 2012 through 2019. I focus on first-time, full-time undergraduate enrollment and enrollment among Pell Grant recipients.

I had two data limitations compared to Howard's original study. I could not obtain Penn State's disaggregated campus-level data for 2019, and I did not have the student migration data needed to recreate his in-state enrollment analysis. I therefore made some changes to the sample and focused on the main difference-in-differences (DiD) and event-study findings.

## Data

I pulled the data and completed the cleaning process in my data cleaning notebook. I organized the raw files into three groups:

1. **IPEDS data:** Enrollment, Pell Grant recipients, tuition and fees, and institutional characteristics from the Integrated Postsecondary Education Data System.
2. **Bureau of Labor Statistics (BLS) data:** County-level unemployment rates.
3. **County FIPS data:** A county-name and FIPS-code file shared by an OpenDataPhilly user. I used this to connect the county names in my IPEDS files to the FIPS codes needed for the BLS merge. The IPEDS extracts I downloaded provided county names without the codes I needed.

| Source | Data I used | Years |
| --- | --- | --- |
| [IPEDS, National Center for Education Statistics](https://nces.ed.gov/ipeds/use-the-data) | Institution identifiers, enrollment counts, Pell Grant counts, tuition and fees, institutional control, and location | Fall 2012–2019 |
| [BLS, Local Area Unemployment Statistics](https://www.bls.gov/lau/) | County unemployment rates | 2011–2018, lagged one year |
| OpenDataPhilly user-contributed county FIPS file | County names and FIPS codes used to match IPEDS and BLS | County crosswalk |

For the analysis, my unit of observation is an **institution-year**. New York public institutions make up the treatment group, and Pennsylvania public institutions make up the comparison group. I use 2012–2019, with Excelsior introduced in 2017.

My final sample has **111 public institutions**, including **74 in New York and 37 in Pennsylvania**, observed over eight years (Different than Howard because he had 125 institutions total, and 51 in Pennsylvania).

### Main Variables

| Variable | How I use it |
| --- | --- |
| `unitid` | Identify institutions, add institution fixed effects, and cluster standard errors |
| `fall_year` | Identify the year and add year fixed effects |
| `state` | Distinguish New York from Pennsylvania |
| `Grand total` | First-time, full-time undergraduate enrollment |
| `Number of full-time first-time undergraduates awarded Pell grants` | First-time, full-time Pell Grant enrollment |
| `public` | Keep public institutions |
| `Published in-state tuition and fees` | Create the logged tuition variable for adjusted models |
| `unemp_lag` | Control for county unemployment in the preceding year |

These are the main variables I use in the analysis notebook.

## Data Cleaning

I document my cleaning process in [`Data_cleaning.ipynb`](Data_cleaning.ipynb). I started with separate annual files and worked toward one institution-year panel.

1. **Merging IPEDS files.** I merged the annual Pell Grant, race and gender enrollment, and tuition files using `unitid`. I also standardized the column names across years.
2. **Creating the panel.** I combined the annual datasets into one file and added `fall_year`.
3. **Adding county FIPS codes.** I matched the IPEDS county names to the county FIPS crosswalk sourced from an OpenDataPhilly user.
4. **Merging unemployment data.** I cleaned the BLS files and matched unemployment rates to institutions using county FIPS codes. I lagged unemployment by one year (`unemp_year = fall_year - 1`).
5. **Keeping public institutions.** I used the IPEDS institutional-control variable to identify public colleges and universities.
6. **Applying exclusions.** I excluded specialty institutions identified in Howard's dissertation and removed Penn State campuses because of the 2019 data limitation.
7. **Saving the cleaned dataset.** I exported the final panel as `panel_clean.csv` for the analysis notebook.

### Why I Excluded Penn State

Beginning in 2019, Penn State changed how it reported campus enrollment data in IPEDS. Howard obtained supplemental campus-level data directly from Penn State, but I did not have access to those records. So I chose to remove all Penn State campuses so I could keep a consistent set of institutions across years without filling in missing values.

This is the exclusion I use for my final sample:

```python
psu = panel["institution name"].str.startswith(
    "Pennsylvania State University",
    na=False
)

panel = panel.loc[~psu].copy()
```

The exclusion changes the Pennsylvania comparison group, so my results will not match Howard's estimates exactly.

## Analysis

I document the analysis in [`Excelsior_Replication_Step_by_Step.ipynb`](Excelsior_Replication_Step_by_Step.ipynb). I follow Howard's main public-institution specifications and compare my results to Tables 10–13 in his dissertation.

### 1. Checking Parallel Trends

I first plot average enrollment in New York and Pennsylvania between 2012 and 2019. I then test whether the two states had significantly different linear enrollment trends before treatment.

For the statistical test, I interact the New York indicator with a linear time trend and include institution and year fixed effects. I run the test with treatment beginning in both 2017 and 2018.

None of my pretreatment slope tests were statistically significant at the 5% level. I therefore fail to reject the null hypothesis that the pretreatment slopes are equal, which provides some support for the parallel trends assumption. 

### 2. Difference-in-Differences

Next, I estimate two-way fixed-effects DiD models:

$$
Y_{it} = \alpha + \gamma_i + \lambda_t
+ \beta(\text{NY}_i \times \text{Post}_t) + \varepsilon_{it}
$$

Here, $Y_{it}$ is enrollment at institution $i$ in year $t$, while $\gamma_i$ and $\lambda_t$ represent institution and year fixed effects. The coefficient $\beta$ captures the estimated change in New York enrollment relative to Pennsylvania after treatment.

I run separate models with treatment starting in **2017** and **2018**. I start without covariates, then add logged in-state tuition and county unemployment lagged one year. I cluster standard errors at the institution level.

### 3. Event Study

Lastly, I estimate enrollment differences separately for each year, using **2016 as the reference year**:

$$
Y_{it} = \alpha + \gamma_i + \lambda_t
+ \sum_{k \ne 2016} \delta_k
(\text{NY}_i \times \mathbf{1}[t=k])
+ \varepsilon_{it}
$$

I use the estimates from 2012–2015 to examine pretreatment differences and the estimates from 2017–2019 to examine changes following Excelsior's introduction. I also plot the coefficients with 95% confidence intervals.

The event-study models reported below include institution and year fixed effects and institution-clustered standard errors (they don't include the additional tuition and unemployment covariates.)

## Main Findings

### Difference-in-Differences Results

After adding tuition and lagged county unemployment, my estimated treatment coefficients decreased across all four models. The coefficients remained positive, although none were statistically significant at the 5% level.

| Outcome | Treatment begins | Without covariates | With covariates | Adjusted p-value |
| --- | ---: | ---: | ---: | ---: |
| First-time, full-time enrollment | 2017 | 82.99 | 38.26 | 0.444 |
| Pell Grant enrollment | 2017 | 50.64 | 27.14 | 0.319 |
| First-time, full-time enrollment | 2018 | 85.78 | 58.33 | 0.196 |
| Pell Grant enrollment | 2018 | 46.11 | 30.30 | 0.238 |

Overall, I find a similar pattern to Howard's DiD results. The estimates suggest increases in enrollment, but the adjusted models have substantial uncertainty.

### Event-Study Results

My event-study estimates show positive enrollment changes in New York following Excelsior's introduction.

| Year | My FTFT estimate | Howard's FTFT estimate | My Pell FTFT estimate | Howard's Pell FTFT estimate |
| --- | ---: | ---: | ---: | ---: |
| 2017 | **63.65** (p = 0.004) | 68.35 | **43.97** (p = 0.001) | 51.38 |
| 2018 | **100.01** (p = 0.002) | 83.29 | **56.69** (p = 0.005) | 51.95 |
| 2019 | **102.05** (p = 0.017) | 79.21 | 48.84 (p = 0.058) | 44.90 |

For first-time, full-time undergraduate enrollment, all three post-treatment coefficients were statistically significant at the 5% level. The estimates suggest increases of approximately 64, 100, and 102 students per New York public institution in 2017, 2018, and 2019, respectively, relative to Pennsylvania.

I observed similar results for Pell Grant recipients. The estimated increases were approximately 44 students in 2017, 57 in 2018, and 49 in 2019. The 2017 and 2018 coefficients were significant at the 5% level, while the 2019 estimate was marginally significant at the 10% level.

None of the individual pretreatment event-study coefficients were statistically significant at the 5% level. I still need to report the joint pretreatment test alongside these estimates.

My results are generally consistent with Howard's main event-study findings. The differences in coefficient magnitudes may be related to my exclusion of Penn State and other differences in how the data were constructed.

## Repository Structure

The notebooks use the following file paths. I keep the original data files organized by source and save the final cleaned panel for the analysis.

```text
.
├── README.md
├── Data_cleaning.ipynb
├── Excelsior_Replication_Step_by_Step.ipynb
├── IPEDS Data/
│   ├── Pell Grant/
│   ├── Race/
│   └── Tuition and Fees/
├── BLS Data/
├── Other Data/
│   ├── county_fips_master.csv
│   ├── 2019_county.csv
│   ├── 2012_control.csv
│   └── 2019_control.csv
└── panel_clean.csv
```

The IPEDS and BLS folders contain the annual source files used in the cleaning notebook. Some filenames are specific to my downloads, so the local paths need to match those in the code.

## Reproducing My Analysis

To run the notebooks, I use Python with pandas, NumPy, Matplotlib, statsmodels, and openpyxl. The following command installs the main packages:

```bash
python -m pip install pandas numpy matplotlib statsmodels openpyxl notebook
```

After placing the raw IPEDS, BLS, and county FIPS files in the corresponding folders, I run `Data_cleaning.ipynb` to generate `panel_clean.csv`. I then run `Excelsior_Replication_Step_by_Step.ipynb` to estimate the DiD and event-study models.

Before comparing results, I check that my sample contains 74 New York and 37 Pennsylvania institutions in each year. I use Howard's Tables 10–13 as my benchmarks.

## Limitations

My analysis has several limitations that are important for interpreting the findings.

- **Penn State:** I excluded all Penn State campuses because I did not have the disaggregated 2019 data Howard used. This changes the Pennsylvania sample and limits how closely I can reproduce his estimates.
- **Student migration:** I did not recreate Howard's in-state enrollment models because I did not have the migration data needed for those specifications.
- **Tuition:** I use the natural logarithm of published in-state tuition and fees in my adjusted models. I have not confirmed whether this is the exact tuition measure Howard used.
- **Causal interpretation:** My analysis depends on the parallel trends assumption and the absence of other shocks that affected the two states differently. The pretreatment tests provide some support, but they cannot establish those assumptions.
- **Scope:** I focused on the main public-institution findings. I did not recreate every outcome, institutional sector, or additional analysis in the dissertation.

## Reference

Howard, J. E. (2026). *Excelsior scholarship and college enrollment in New York: Evidence of unexpected gains* [Doctoral dissertation, University of Pennsylvania].

I use Howard's dissertation as the basis for the research design and as the benchmark for comparing the results. I completed the data cleaning and Python replication independently.
