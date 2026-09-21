# Coding Guidelines — Environmental Epidemiology Analysis in R

This document defines the coding conventions for this project. All contributors are expected to follow these guidelines to ensure consistency, reproducibility, and readability across the codebase.

---

## Table of Contents

1. [Variable Naming](#1-variable-naming)
2. [Code Annotation](#2-code-annotation)
3. [Readability](#3-readability)
4. [Avoid Hardcoding](#4-avoid-hardcoding)
5. [Project Structure](#5-project-structure)
6. [Reproducibility](#6-reproducibility)
7. [Data Handling](#7-data-handling)
8. [Packages](#8-packages)
9. [Version Control Etiquette](#9-version-control-etiquette)

---

## 1. Variable Naming

### Use `janitor::clean_names()` on all incoming data

Every raw dataset must be passed through `janitor::clean_names()` immediately after loading. This converts column names to `snake_case`, removes special characters, and ensures consistency regardless of the source format.

```r
library(janitor)
library(readr)

raw_data <- read_csv(file.path(onedrive_raw, "exposure_measurements.csv")) %>%
  clean_names()
```

### Use `snake_case` throughout

All variable names, function names, and file names use `snake_case`. This is consistent with `janitor` output and the `tidyverse` style.

```r
# ✅ Good
pm25_daily_mean    <- ...
hospital_admissions <- ...
study_population   <- ...

# ❌ Avoid
PM25DailyMean      <- ...
HospitalAdmissions <- ...
studyPopulation    <- ...
```

### Name variables to convey meaning and context

Variable names should describe **what** the variable contains — including units or time scale where helpful.

```r
# ✅ Good
temp_celsius_daily       <- ...
lag_days_exposure        <- 3
o3_ppb_max_8hr           <- ...
death_cause_cardiovascular <- ...

# ❌ Avoid
temp    <- ...
lag     <- 3
x       <- ...
data2   <- ...
```

### Common naming conventions for this project

| Concept | Convention | Example |
|---|---|---|
| Exposure variables | `[pollutant]_[unit]_[aggregation]` | `pm25_ug_daily_mean` |
| Outcome variables | `outcome_[cause]` | `outcome_resp_admission` |
| Lag variables | `[var]_lag[n]` | `pm25_ug_daily_mean_lag3` |
| Index/ID columns | `[entity]_id` | `county_id`, `patient_id` |
| Date/time columns | `date_[description]` | `date_admission`, `date_exposure` |
| Flags/indicators | `is_[condition]` or `flag_[condition]` | `is_missing`, `flag_extreme_value` |
| Derived/modelled outputs | `pred_[description]` | `pred_mortality_rate` |

### Avoid reusing names

Do not reuse a variable name for a different purpose within the same script. Use distinct, descriptive names at every step of a pipeline.

```r
# ❌ Avoid — overwriting the same variable repeatedly
df <- read_csv("data.csv")
df <- clean_names(df)
df <- df %>% filter(!is.na(pm25))

# ✅ Preferred — trace the pipeline clearly
raw_exposure     <- read_csv(file.path(onedrive_raw, "exposure.csv")) %>% clean_names()
exposure_clean   <- raw_exposure %>% filter(!is.na(pm25_ug_daily_mean))
```

---

## 2. Code Annotation

### Every `.Rmd` file starts with a YAML header and setup chunk

Use the YAML front matter and a setup chunk at the top of every R Markdown file.

```markdown
---
title: "02 — Merge Exposure and Outcomes"
author: "[Your name]"
date: "`r Sys.Date()`"
output: html_document
---
```

Followed immediately by a setup chunk:

```r
#```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, warning = FALSE, message = FALSE)

library(tidyverse)
library(janitor)
library(here)
library(lubridate)
#```
```

### Use descriptive chunk names and prose between chunks

Every code chunk should have a meaningful name. Use the prose sections between chunks to explain what is happening and why.

```markdown
Data are restricted to the warm season (May–September) based on prior literature
showing stronger PM2.5–mortality associations during summer months.

[code chunk: filter-warm-season]
analysis_data <- analysis_data %>%
  filter(month(date_exposure) %in% 5:9)
```

### Use Markdown headers to organize the document

Structure each `.Rmd` with headers that reflect the analytical flow.

```markdown
## 1. Load and inspect data
## 2. Clean and validate
## 3. Merge datasets
## 4. Compute lag variables
## 5. Session info
```

### Flag assumptions, data quirks, and known limitations in prose

Use blockquotes or bold text in the Markdown narrative — not just inside code comments.

```markdown
> **Note:** County FIPS 06037 (Los Angeles) has missing monitor data for
> June–August 2018. Imputed values are used for this period; interpret
> results with caution.

**TODO:** Replace the manual date range below with a config-driven approach
once the data pipeline is finalized.
```

---

## 3. Readability

### Use the tidyverse pipe `%>%`

Use `%>%` from `magrittr` (loaded automatically with `tidyverse`). Break chains across lines so each step is visible.

```r
# ✅ Good
analysis_data <- raw_data %>%
  clean_names() %>%
  filter(!is.na(pm25_ug_daily_mean)) %>%
  mutate(log_pm25 = log(pm25_ug_daily_mean + 1)) %>%
  left_join(population_data, by = "county_id")

# ❌ Avoid
analysis_data <- left_join(mutate(filter(clean_names(raw_data), !is.na(pm25_ug_daily_mean)), log_pm25 = log(pm25_ug_daily_mean + 1)), population_data, by = "county_id")
```

### One chunk, one purpose

Each code chunk in an `.Rmd` should do one logical thing — loading data, cleaning, a single model, a single figure. Split large chunks rather than grouping unrelated steps together.

### Keep lines under 80 characters

```r
# ✅ Good — break long function calls across lines
model_out <- glm(
  outcome_resp_admission ~ pm25_ug_daily_mean + temp_celsius_daily +
    relative_humidity_pct + day_of_week + ns(date_exposure, df = 8),
  data   = analysis_data,
  family = poisson(link = "log")
)

# ❌ Avoid
model_out <- glm(outcome_resp_admission ~ pm25_ug_daily_mean + temp_celsius_daily + relative_humidity_pct + day_of_week + ns(date_exposure, df = 8), data = analysis_data, family = poisson(link = "log"))
```

### Let prose do the explaining, not inline comments

In R Markdown, the narrative between chunks is the primary place for explanation. Reserve in-chunk comments for short technical notes that don't belong in prose.

```markdown
We use a quasi-Poisson model to account for overdispersion in the daily
admission counts. The time trend is controlled with a natural spline (8 df/year).

[code chunk: model-primary]
model_out <- glm(...)
```

### Prefer explicit, verbose code over terse clever code

Readability beats brevity. Write code that a collaborator can understand on first reading.

```r
# ✅ Clear
high_exposure_days <- analysis_data %>%
  filter(pm25_ug_daily_mean > quantile(pm25_ug_daily_mean, 0.75, na.rm = TRUE))

# ❌ Terse — what does this do at a glance?
hed <- analysis_data[analysis_data$pm25 > quantile(analysis_data$pm25, .75, na.rm = T), ]
```

---

## 4. Avoid Hardcoding

### Define all parameters at the top of the script or in a config file

Never bury numeric values or file paths inside the body of the code. Define them once at the top so they are easy to find and change.

```r
# ==============================================================================
# Configuration
# ==============================================================================

study_start_date   <- as.Date("2010-01-01")
study_end_date     <- as.Date("2020-12-31")
warm_season_months <- 5:9          # May through September
lag_max            <- 7            # Maximum lag (days)
spline_df_time     <- 8            # Degrees of freedom for time spline
quantile_threshold <- 0.75         # Threshold for high-exposure flagging

# Data paths — see "Locating project data" below. onedrive_path is the one
# deliberate exception to "avoid hardcoding" here: it's machine-specific,
# so edit it to match where OneDrive syncs on your computer.
onedrive_path      <- "C:/Users/yourname/OneDrive - URBE Lab/Projects/this-project"
onedrive_raw       <- file.path(onedrive_path, "raw")
onedrive_processed <- file.path(onedrive_path, "processed")

path_exposure   <- file.path(onedrive_processed, "exposure_clean.rds")
path_admissions <- file.path(onedrive_processed, "admissions_clean.rds")
path_output     <- file.path(onedrive_processed, "analysis_dataset.rds")
```

### Use `here::here()` for all paths inside the repository

Never use absolute paths for anything that lives in this repo (scripts, `outputs/`, `reports/`). `here()` builds paths relative to the project root, making the code portable across machines and operating systems.

```r
library(here)

source(here("R", "01_clean_exposure.R"))
saveRDS(figure_data, here("outputs", "tables", "summary_table.rds"))
```

### Locating project data (OneDrive)

Data never lives inside this repository — see [Project Structure](#5-project-structure) and [Data Handling](#7-data-handling). The OneDrive sync path is different on every machine and OS, so each script sets its own `onedrive_path` at the top, in its Configuration block, as shown above. Before running a script, update that one line to match where OneDrive syncs on your computer.

**Be careful with commits.** Because `onedrive_path` is a plain line in the script, git will track it like any other change. Check `git diff` before committing — don't bundle your personal path with real code changes, and don't commit a path that only makes sense on your machine unless you're deliberately updating it for the whole team (e.g. after the lab reorganizes the OneDrive folder).

### Keep model specifications readable and reusable

```r
# Define model formula separately so it can be inspected and reused
model_formula <- outcome_resp_admission ~
  pm25_ug_daily_mean +
  temp_celsius_daily +
  relative_humidity_pct +
  day_of_week +
  ns(date_exposure, df = spline_df_time)

model_out <- glm(model_formula, data = analysis_data, family = poisson(link = "log"))
```

---

## 5. Project Structure

Organize files using the following layout. **Data is never stored in the repository** — see [Data Handling](#7-data-handling) for where it actually lives.

```
project/
├── R/
│   ├── 01_clean_*.R    # Data cleaning scripts
│   ├── 02_merge_*.R    # Data merging scripts
│   ├── 03_analysis_*.R # Statistical analyses
│   └── 04_figures_*.R  # Output figures and tables
├── reports/            # R Markdown reports (.Rmd)
├── outputs/
│   ├── figures/
│   └── tables/
├── CODING_GUIDELINES.md
├── README.md
└── project.Rproj
```

Number scripts in execution order. Use descriptive suffixes (e.g., `01_clean_exposure.R`, `01_clean_admissions.R`).

---

## 6. Reproducibility

### Set a random seed wherever randomness is involved

```r
set.seed(2024)
```

### Document your R and package versions

Call `sessionInfo()` at the end of key R Markdown reports to record the environment used for that analysis.

```r
sessionInfo()
```

---

## 7. Data Handling

### Data lives in OneDrive, not in this repository

Raw data is pulled from the lab's shared OneDrive folders. Each project has its own dedicated OneDrive folder for processed/merged data. Nothing under `data/` — raw or processed — is ever part of the git repository, tracked or ignored; see [Locating project data](#locating-project-data-onedrive) for how scripts find it.

### Never modify raw data

Treat raw data pulled from OneDrive as read-only. All transformations happen in R scripts, with results written back to the project's OneDrive processed-data folder (via `onedrive_processed`, not into the repo).

### Validate data after loading and merging

Always check dimensions, key variable distributions, and merge results explicitly.

```r
# After loading
stopifnot(nrow(raw_data) > 0)
stopifnot(all(!is.na(raw_data$county_id)))

# After merging
cat("Rows before merge:", nrow(exposure_clean), "\n")
cat("Rows after merge: ", nrow(analysis_data), "\n")
cat("Matched counties: ", n_distinct(analysis_data$county_id), "\n")
```

### Handle missing data explicitly and document decisions

Do not silently drop missing data. Name what is being excluded and why.

```r
# Removing days with missing PM2.5 values — see data quality report
# (reports/00_data_quality.html). Represents < 1% of study days.
exposure_clean <- raw_exposure %>%
  filter(!is.na(pm25_ug_daily_mean))
```

---

## 8. Packages

### Load packages at the top of every script

All `library()` calls go at the very top of each script or `.Rmd` file — never buried inside functions or analysis blocks. There is no shared setup file; each script is self-contained and explicit about its dependencies.

```r
library(tidyverse)    # Data manipulation and visualization
library(ggplot2)      # Figures (loaded by tidyverse, listed explicitly for clarity)
library(janitor)      # Column name cleaning
library(here)         # Portable file paths
library(lubridate)    # Date handling
library(tableone)     # Descriptive statistics / Table 1
```

Only load packages that the script actually uses. Remove unused `library()` calls.

### Use `::` for clarity when namespace conflicts are likely

```r
# Explicit to avoid conflict with dplyr::filter vs stats::filter
dplyr::filter(data, !is.na(pm25_ug_daily_mean))
```

---

## 9. Version Control Etiquette

### Write clear commit messages

Each commit message should complete the sentence: *"This commit will..."*

```
# ✅ Good
Add lag structure to exposure dataset (lags 0–7)
Fix duplicate rows in admissions merge
Update time spline df from 6 to 8 per reviewer comments

# ❌ Avoid
fix
update
changes
wip
```

### Commit one logical change at a time

Don't bundle unrelated changes in a single commit. This makes it easier to review changes and revert if needed.

### Never commit sensitive data or credentials

Data is never in the repo to begin with (see [Data Handling](#7-data-handling)), but the `.gitignore` also blocks it as a safety net in case a script accidentally writes an export into the repo folder:

```
# .gitignore
.Renviron
outputs/**/*
!outputs/**/.gitkeep
*.rds
*.csv
renv/library/
```

If a small reference/lookup table genuinely belongs in the repo (e.g. a FIPS code crosswalk), force it in explicitly with `git add -f` rather than loosening the blanket rule above.

---


