COVID-19 Case Study Documentation
=================================

**Overview:** This case study analyzes global COVID-19 data (confirmed cases, deaths, recoveries) over time. It uses Python (pandas, matplotlib) to clean, transform, and merge data, then examines pandemic trends across countries. The goal is to understand how case counts evolved, identify peaks and rates, and draw insights on the pandemic’s impact.

.. contents::
   :depth: 2
   :local:

Dependencies & Setup
--------------------

The analysis uses Python (>=3.6) with libraries including pandas, numpy, and matplotlib.  Example setup steps (in code):

**Q-1. Data Loading:**

::

    import pandas as pd
    import numpy as np
    import matplotlib.pyplot as plt
    covid_confirmed = pd.read_csv('covid_19_confirmed.csv')
    covid_deaths = pd.read_csv('covid_19_deaths.csv')
    covid_recovered = pd.read_csv('covid_19_recovered.csv')

The CSV datasets contain cumulative COVID-19 statistics per region by date. They are likely from the Johns Hopkins CSSE repository (as used by many analysts)【13†L96-L100】. No special APIs are required; just reading the provided CSV files.

Data Description
----------------

- `covid_19_confirmed.csv`: A wide table of cumulative confirmed cases. Columns: `Province/State`, `Country/Region`, `Lat`, `Long`, followed by dates (1/22/20 onwards). Each row is a region (province or country) and values are total confirmed cases up to each date. This is used to compute new cases, peaks, and trends (often by summing per country).

- `covid_19_deaths.csv`: Same structure as confirmed, but with cumulative deaths by date. It has an extra header row that is removed; otherwise, it is grouped by country to get country-level death counts.

- `covid_19_recovered.csv`: Same structure, with cumulative recoveries. Also contains an extra header row to remove. It is merged with the above after cleaning to compare recoveries vs confirmed cases.

In all three, date columns are parsed to datetime when needed. The data often gets transformed (melted) so that each (Country, Date) pair is a single row.

Methodology / Analysis Steps
----------------------------

**Q-2. Data Exploration:** The notebook begins by examining each dataset’s structure (`.head()`). Key points:
- All three files share the same columns: `Province/State, Country/Region, Lat, Long, date1, date2, ...`. Usually `Country/Region` has a name and `Province/State` is blank for country-level totals.
- `Lat/Long` are floats; date columns hold integer counts. Many entries have NaN in `Province/State` (if data is at the country level).
- The deaths and recovered CSVs have an extra header line, which is removed by resetting the header to the first data row.
- After cleaning, each table has no extra NaNs except in `Province/State`.

Next, global confirmed cases trends are analyzed for top countries (Section 2.2):
- Date columns are converted to datetime objects. The code finds the last day of each month (`month_end`) by grouping dates by year/month and taking `.max()`.
- Confirmed cases are grouped by `Country/Region` and summed (aggregating provinces): `cases_per_country = covid_confirmed.groupby('Country/Region').sum()`.
- Top 7 countries by total cases on the latest date are identified: `top_7_countries = cases_per_country[latest_date].sort_values(ascending=False).head(7).index`.
- A new DataFrame contains these countries’ cumulative cases at month-end (`month_end_cases_top_7_countries`).
- A line plot is created for all 7 countries over time, and individual subplots for each country. This illustrates how each country’s cases grew over time (e.g., USA and India show the largest curves).

China-specific exploration (Section 2.3):
- Filter confirmed data for `Country/Region == 'China'`. Sort provinces by total cases and take top 5 (Hubei, Hong Kong, Guangdong, Shanghai, Heilongjiang).
- Build `month_end_cases_top_5_chinese_provinces` for those provinces’ month-end case counts, and `total_cases_china_month_end` for China overall.
- Plot China’s total curve and each of the top 5 provinces. Hubei’s case count is orders of magnitude higher than others, and all other provinces (not plotted) had very few cases. This shows China’s outbreak was concentrated in one region.

**Q-3. Handling Missing Data:** The notebook checks for nulls in each dataset. It finds NaNs in `Province/State`, `Lat`, `Long` (expected for country-level entries), and in one date column (`4/20/20`) in deaths/recovered:
- *Confirmed data:* Two rows have NaN lat/long: “Repatriated Travellers (Canada)” and “Unknown (China)”. These are filled by computing the median (or mean) latitude/longitude of the corresponding country’s other entries and assigning those values.
- *Deaths data:* The same two regions have NaNs. The code copies the confirmed-case values for lat/long into the deaths table.
- *Recovered data:* One NaN lat/long row (“Unknown, China”) is filled by the mean lat/long of other Chinese provinces.
- In deaths and recovered, the date `4/20/20` had a NaN for one country (Algeria in deaths, Argentina in recovered). The solution is to forward-fill across dates for that row, replacing the NaN with the previous day’s value.
- After this, only `Province/State` has NaNs (meaning “all country”). 

**Q-4. Data Cleaning and Preparation:** Any remaining missing `Province/State` entries are filled with “All Provinces” (using `fillna({'Province/State':'All Provinces'})`). A final check shows no NaNs remain. Now each dataset is ready for analysis.

**Q-5. Independent Data Analysis:** Two questions are answered:
- *Peak daily new cases (Germany, France, Italy):* For each country, confirmed cases are summed across provinces, then daily new cases are computed via `.diff()`. The peak is the maximum daily increase. Results (in table form): Germany – 49,044 on 2020-12-30; France – 117,900 on 2021-04-11; Italy – 40,902 on 2020-11-13. France’s peak is the highest (117,900).
- *Recovery rates on Dec 31, 2020 (Canada vs Australia):* It extracts confirmed and recovered counts on 12/31/2020 for those countries (summing provinces). The recovery rate = Recovered/Confirmed. It prints a comparison table: Canada’s recovery rate is ~84.47%, Australia’s ~79.38%. Thus Canada had a higher recovery percentage by the end of 2020.

**Q-6. Data Transformations:** The deaths data is reshaped and analyzed (Section 6):
- *6.1 Wide-to-long (Deaths):* Uses `pd.melt` with id_vars `['Province/State','Country/Region','Lat','Long']` and value_vars as all dates. The result `long_deaths` has columns `Date` and `Deaths`. `Date` is converted to datetime and the table is sorted.
- *6.2 Total deaths per country:* It converts the last date column to numeric and groups by `Country/Region` to sum it. This yields `deaths_per_country` with total deaths per country, sorted descending.
- *6.3 Top 5 by avg daily deaths:* All date columns are numeric, then grouped by country (summing) to get country-total time series, then `.diff()` to daily new deaths. Fill the first-day NaNs with the original count. The average of daily new deaths per country is computed, and the top 5 countries are listed (e.g., USA, Brazil, India, etc.).

**Q-7. Data Merging:** All metrics are combined (Section 7):
- A helper function `load_and_melt(df, value_name)` is defined to melt any of the wide tables into long form with columns `[Province/State, Country/Region, Lat, Long, Date, <value_name>]`. It’s applied to confirmed, deaths, and recovered.
- Each melted table is aggregated by country and date (summing provinces): `agg_conf`, `agg_deaths`, `agg_recovered`.
- These are merged together on `(Country/Region, Date)` using outer joins. The result `complete_merge` has `[Country/Region, Date, Confirmed, Deaths, Recovered]` for all dates.
- A fix is applied: the US recovered column had zeros. The code replaces 0 with NaN and forward-fills within each country so that recoveries accumulate properly. Remaining NaNs (initial entries) are set to 0.

**Q-8. Combined Data Analysis:** Using the merged dataset (Section 8):
- *8.1 Highest death-rate countries (2020):* Filter to dates in 2020. Compute death rate = Deaths/Confirmed for each record (dropping any invalid rows). For each country, compute the average death rate over 2020. The top 3 are Yemen (~28.7%), Italy (~14.3%), and Sudan (~13.6%). (Cruise ships like MS Zaandam are excluded.)
- *8.2 South Africa recoveries vs deaths:* Extract South Africa’s data. Using the latest date row, print total deaths and recoveries (and their ratio, ~28 recovered per death). Create a bar chart comparing total recoveries vs deaths, and a line plot of cumulative recoveries and deaths over time. These show South Africa had far more recoveries than deaths (low fatality fraction).
- *8.3 US monthly recovery ratio:* Filter US data from Mar 2020–May 2021, group by `YearMonth` summing Confirmed and Recovered. Compute `RecoveryRatio = Recovered/Confirmed` per month and find the max (in October 2020, ~39.6%). Plot `RecoveryRatio (%)` over time. The notebook notes this October peak likely comes from many recoveries being reported for prior cases, temporarily inflating the ratio.

Results & Visualizations
-----------------------

Key results include:

- *Initial tables:* (`.head()`) confirmed the data format and prompted the cleaning steps above (extra headers and missing values).
- *Top-7 Countries Plot:* A combined line chart of cumulative confirmed cases (month-end) for the Top 7 countries. It clearly illustrates each country’s case growth; for example, USA and India far outpace others. Individual subplots for each country highlight their own trajectories.
- *China & Provinces Plot:* A line plot of China’s total cases per month, plus subplots for its top 5 provinces. Hubei’s curve dwarfs the others; all other provinces (not shown) had very few cases. This highlights how China’s outbreak was concentrated in one province.
- *Peak Cases Table (5.1):* A printed summary of peak daily new cases for Germany, France, Italy (with dates). France’s 117,900 cases on April 11, 2021 is the largest of the three.
- *Recovery Rate Table (5.2):* A small table comparing Dec 31, 2020 totals for Canada and Australia (confirmed, recovered, and % recovered). It shows Canada’s ~84.47% vs Australia’s ~79.38%.
- *Total Deaths (6.2):* A printed list of countries and their total deaths. This identifies the hardest-hit countries by mortality (e.g., USA, Brazil, etc.).
- *Top Avg Daily Deaths (6.3):* A printed list of the top 5 countries by average new daily deaths (e.g., USA, Brazil, India, etc.).
- *Top Death Rates (8.1):* A printed list of the top 3 countries by average death rate (Yemen, Italy, Sudan).
- *South Africa Charts (8.2):* A bar chart of South Africa’s total recoveries vs deaths (recoveries far higher), and a time-series line plot of cumulative recoveries and deaths. Visually, recoveries dominate.
- *US Recovery Ratio Plot (8.3):* A line chart of US monthly recovery percentage from Mar 2020 to May 2021, peaking in Oct 2020. This supports the explanation that October saw delayed recoveries.

From these, insights include: France’s enormous April 2021 spike, China’s outbreak focused in Hubei, and how reporting timing affected the October 2020 recovery ratio.

Conclusions / Takeaways
-----------------------

- The COVID-19 pandemic tested healthcare systems and data infrastructures worldwide.
- Comparing confirmed, death, and recovery trends highlights how response and resources affected outcomes (e.g. very high death rates in Yemen, Italy, Sudan).
- The findings stress the need for robust public health infrastructure, timely reporting, and vaccine access to manage such crises.
- Societal lessons: Even under unprecedented challenges, coordinated responses emerged, underscoring the importance of preparedness for future health emergencies.

References / Further Reading
---------------------------

- Johns Hopkins University CSSE COVID-19 Data Repository【13†L96-L100】 (source of global cases/deaths data).
- Pandas documentation: https://pandas.pydata.org (used for data manipulation).
- Matplotlib documentation: https://matplotlib.org (used for plotting).
- (No external news or papers were directly referenced in the notebook; these resources provide context on the datasets used.)
