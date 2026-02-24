# MyDocument – Learnings, Challenges, Assumptions

## Assumptions

- **Dataset scope**: The NYC jobs CSV is the single source of truth. “Highest-paid skills in the US market” is interpreted as **highest-paid skills in this NYC job postings dataset** (no external US salary data).
- **Last 2 years**: Defined using the maximum `Posting Date` in the data minus 730 days (e.g. if latest posting is 2019-12-17, “last 2 years” is 2017-12-17 to 2019-12-17).
- **Salary comparison**: Salaries are normalized to **annual** for KPIs: Hourly × 2080, Daily × 260; otherwise the range midpoint is used. Missing or invalid salary values are excluded from salary-based KPIs.
- **Degree vs salary (KPI 3)**: “Higher degree” is inferred from text in **Minimum Qual Requirements** using keywords (e.g. baccalaureate, master's, PhD). Correlation is observational (postings that mention higher degrees have higher average salary in this sample).
- **Skills (KPI 6)**: Skills are parsed from **Preferred Skills** by splitting on delimiters (e.g. commas, bullets). Only phrases with 4–80 characters and appearing in at least 5 postings are used to avoid noise.
- **Target path**: Processed output is written to `/notebook/processed_nyc_jobs` (Parquet). In Docker this is under the mounted notebook directory; adjust if running elsewhere.

## Learnings

- **CSV types**: All columns are read as string. Profiling and casting (salary, dates, integers) are necessary before aggregation and feature engineering.
- **Duplicate rows**: The same job can appear as both Internal and External; counts are “posting records” not unique jobs unless deduplicated by `Job ID` (or similar).
- **Long text columns**: Columns like Job Description and Minimum Qual Requirements are useful for NLP or keyword extraction but were dropped in the processed dataset to keep a compact schema; they can be reattached from the raw data if needed.
- **Visualization**: Matplotlib is used for bar charts; results are collected with `.toPandas()` for plotting. For very large data, sampling or pre-aggregation in Spark is preferable to collecting full result sets.

## Challenges

- **Degree extraction**: Minimum Qual Requirements are free text; the degree-level logic is heuristic (regex). Some postings mention multiple degrees (e.g. “bachelor or equivalent”); we map to a single level and may misclassify edge cases.
- **Skills extraction**: Preferred Skills format varies (bullets, commas, line breaks). Splitting and trimming yields mixed-quality “skills”; filtering by length and minimum count improves robustness but may drop valid rare skills.
- **Date handling**: Some date fields are null or malformed; we filter null posting dates where needed (e.g. “last 2 years” KPI) and use Spark’s `to_date` with a single format.

## Considerations

- **Feature removal**: Columns dropped in the processed dataset were chosen to reduce size and focus on structured fields (agency, category, salary, dates, derived degree, posting year). Long text and redundant location/contact fields were removed; they can be kept if downstream use requires them.
- **Deployment**: The solution runs in the provided Docker setup (Jupyter + Spark master/workers). No additional deployment steps are required for the take-home; see below for optional deployment and triggering.

## Deployment (proposal)

- **Containers**: Use `docker compose -f ./docker-compose.yml --project-name my_assesment up -d` to start the cluster and Jupyter.
- **Paths**: Ensure the dataset is mounted (e.g. `./dataset:/dataset`) and the notebook directory is mounted (e.g. `./jupyter/notebook:/notebook`). Processed output path (`/notebook/processed_nyc_jobs`) must be writable by the Jupyter container.
- **Resources**: Per INSTALL.md, at least 8GB RAM is recommended for Spark.

## Triggering the code (suggested approach)

1. **Interactive (notebook)**: Open `assesment_notebook.ipynb` in Jupyter and run all cells (Kernel → Restart & Run All). This runs exploration, KPIs, processing, and tests.
2. **Batch (conversion to script)**: Export the notebook to a `.py` script (e.g. `jupyter nbconvert --to script assesment_notebook.ipynb`) and run with `spark-submit` from the Jupyter/Spark container, e.g.  
   `spark-submit --master spark://master:7077 /notebook/assesment_notebook.py`
3. **Scheduling**: For recurring runs, use a scheduler (e.g. cron or Airflow) to run the script or a wrapper that starts the cluster (if needed), runs the job, and optionally shuts down the cluster.
