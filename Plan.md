# Plan: Did Removing Free Allowances Speed Up Decarbonization in the EU ETS Power Sector?

## Question

The EU Emissions Trading System (EU ETS) caps CO2 emissions from covered
installations and requires them to hold one allowance (EUA) per tonne
emitted. The cap has not tightened evenly across sectors: the **power
sector** lost almost all free allowances around 2013 and has had to buy
nearly all of its allowances at market price since then, while **heavy
industry** (steel, cement, chemicals, refining) still receives
substantial free allocation today, to avoid "carbon leakage" (firms
relocating production to places with no carbon price).

**Question:** Did removing free allowances from the power sector in
~2013 lead to a steeper decline in verified emissions than in
industrial sectors that kept their free allocation?

**Expectation:** Power sector emissions fall faster after 2013, relative
to industrial sectors, because full exposure to the carbon price is a
stronger incentive to switch away from coal than a partially-shielded
price signal.

**What would contradict this:** if industrial emissions fell at a
similar rate despite keeping free allowances. That would suggest the
carbon price isn't the main thing driving behavior here, and something
else (efficiency gains, falling electricity/gas demand, other
regulation) is doing the work instead. That result would matter just as
much as a confirming one, since it would be evidence that free
allocation phase-outs are not, on their own, the lever policymakers
think they are.

## Data Source

**Source:** European Environment Agency (EEA), EU ETS data viewer,
drawing on the EU Transaction Log / Union Registry. Aggregated data by
country, main activity type, and year on verified emissions, allowances
allocated for free, and surrendered units, for all stationary
installations covered by the EU ETS, 2005–2025.

**How it's pulled:** This is not a parameter-based REST API like FRED.
It is a bulk data file published periodically at a stable URL. The
notebook pulls it with `requests.get()` directly into pandas — nothing
is manually downloaded or stored as a local CSV in the repo, so the
pull is still fully automated and reproducible by anyone who re-runs
the notebook. (Noting this up front in case a stricter definition of
"API" is expected — happy to swap this for a different source if so.)

**Dataset:** `eea_t_eu-emission-trading-scheme_p_2005-2025_v03_r00`
(Edition 03.00, published Sept 2026, confirmed 2005–2025 coverage)

- Landing page / metadata record (reference only, not fetched in code):
  https://sdi.eea.europa.eu/catalogue/srv/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda
- Direct file to pull in the notebook (confirmed as the official
  distribution link in the metadata record):
  https://sdi.eea.europa.eu/datashare/s/aZ7Ti5HkRGEbF62/download
  — this returns a ~7.8 MB zip, not a CSV directly. The notebook fetches
  it with `requests.get(url)`, then reads the CSV out of the zip in
  memory (`zipfile` + `io.BytesIO`), so nothing is manually downloaded
  or committed to the repo as a static file.
- EEA republishes this periodically under a new version/URL, so the
  link should be re-checked if the notebook is re-run much later and
  the request fails.

**Codebook / documentation** (needed to interpret `size`,
`citl_information`, `active_installation`, and `main_activity_code` —
not fetched for the analysis itself, just used to write the cleaning
code correctly):
- Activity code mapping (power vs. industry grouping depends on this):
  `Translation of activity codes May 2019.xlsx`
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/Translation%20of%20activity%20codes%20May%202019.xlsx
- EU ETS data viewer user manual (PDF) — expected to define the `size`
  metric categories:
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/EEA_EUETS_data_viewer_user%20manual_June12.pdf
- Background note (PDF, Sept 2026) — methodology, including what
  `citl_information` and `active_installation` represent:
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/EU%20Emission%20Trading%20System%20data%20viewer%20Background%20note%20(september%202026).pdf
- Data quality note (PDF) — known issues by country/year, worth a skim
  before trusting any single cell too far:
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/ETC-CM%20EU-ETS%20data%20quality%20September%202026.pdf

**Actual columns in the data file** (confirmed by opening the zip):
`version`, `country_code`, `main_activity_code`, `active_installation`,
`citl_information`, `year`, `size`, `value`, `unit`.

This is a long/tidy table, not one with separate columns per metric.
`size` holds the metric type (expected to include something like
"Verified emissions" and "Freely allocated allowances" — to be
confirmed by inspection), and `value` + `unit` hold the number and its
unit for that row. `active_installation` and `citl_information` are
flags whose exact meaning needs to be confirmed from the README and the
background-note PDF included in the zip, but `citl_information` is
expected to mark the pre-2013 estimated/harmonized years EEA mentions in
its documentation, and `active_installation` is expected to mark
whether that installation was still operating in that year.

The zip also includes an Excel worksheet and PDF documents, one of
which should be EEA's activity-code translation table — needed to map
`main_activity_code` into power vs. industry groups.

## Cleaning Steps

1. Pull the zip via `requests`, extract the main data file (CSV) in
   memory, and load it into pandas. Also extract the README and the
   activity-code lookup (Excel or PDF) for reference.
2. Inspect and print the unique values of `size`, `citl_information`,
   `active_installation`, and `main_activity_code` before writing any
   filtering logic — the exact category labels need to be confirmed
   from the real data rather than assumed.
3. Filter `size` down to the specific metrics needed: verified
   emissions and freely allocated allowances (exact label names to be
   confirmed in step 2).
4. Decide how to handle `active_installation` and `citl_information`
   based on what they turn out to mean — most likely, keep active
   installations only, and keep the pre-2013 estimated years but flag
   them as a caveat rather than dropping them, since the project
   deliberately covers that period.
5. Drop aviation and maritime activity types (added to the scheme in
   2012 and 2024 respectively) — keep stationary installations only, so
   the time series isn't distorted by scope changes.
6. Map `main_activity_code` into two groups using the activity-code
   lookup table found in the zip:
   - **Power/combustion** (electricity and heat generation)
   - **Industry** (steel, cement, chemicals, refining, pulp & paper,
     etc.)
7. Pivot the filtered long table so each row is `country_code x
   main_activity_code x year`, with separate columns for verified
   emissions and free allocation (pivoting on `size`, values from
   `value`).
8. Restrict to EU member states reporting across the full 2005–2025
   window, to keep the panel balanced.
9. Aggregate emissions and free allocation by `year x group` (sum across
   countries), and also keep a `year x country x group` version for a
   robustness check.
10. Index emissions to a base year (2005 = 100) for each group, so the
    two series are comparable regardless of their different starting
    sizes.
11. Compute free allocation as a share of verified emissions, by group
    and year, to show the actual divergence in free-allowance exposure
    between the two sectors.
12. Confirm `unit` is consistent (expected tCO2 throughout) before
    combining rows — do not assume this without checking.
13. Check for missing years or country gaps and document them rather
    than silently dropping data.

## Charts and Tests

**Chart 1 — Free allocation as % of emissions, power vs. industry,
2005–2025.** Shows the "treatment" directly: confirms power's free
allocation collapsed around 2013 while industry's stayed high. This is
the setup chart — it establishes that the two groups experienced
genuinely different policy exposure.

**Chart 2 — Indexed verified emissions (2005 = 100), power vs. industry,
2005–2025.** The main result chart. If the expectation holds, the power
line should bend down more steeply than the industry line after 2013.

**Test — OLS regression with a year × sector interaction:**

```
log(emissions) ~ year + sector + year:sector
```

estimated separately pre- and post-2013, or with a post-2013 indicator
interacted with sector, to test whether the emissions trend actually
differs between the two groups in a statistically meaningful way,
rather than just eyeballing two lines on a chart.

**Robustness check:** re-run the same comparison using the
country-level panel (clustering or at least checking by country) to
make sure the EU-wide result isn't being driven by one or two large
countries (e.g., Germany or Poland's coal-heavy power sectors).

## Limitations to Note in the Conclusion

- This is correlational, not causal — other things changed after 2013
  too (shale gas and renewables got cheaper, electricity demand
  patterns shifted, national coal phase-out policies arrived on
  different timelines in different countries).
- EEA's activity-type classification changed somewhat between the first
  two trading periods and the current one; the grouping into
  power/industry is a simplification.
- Verified emissions data typically lag by a year or two, so the most
  recent 1–2 years in the dataset may be provisional or incomplete.
- No price data is used here (EU ETS allowance price APIs are mostly
  paywalled or require a separate account), so this tests the effect of
  a *policy design change* (removal of free allocation), not the
  carbon price level itself.
