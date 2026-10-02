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
is manually downloaded or stored as a local data file in the repo, so the
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
  — this returns a ~8.2 MB zip. **The data inside is an Excel workbook
  (`ETS_Database_September_2026.xlsx`, one sheet), not a CSV.** The
  notebook fetches it with `requests.get(url)`, then reads the workbook
  out of the zip in memory (`zipfile` + `io.BytesIO` +
  `pd.read_excel`, with `main_activity_code` read as text), so nothing
  is manually downloaded or committed to the repo as a static file.
  Implemented and verified in the notebook (section 1).
- EEA republishes this periodically under a new version/URL, so the
  link should be re-checked if the notebook is re-run much later and
  the request fails.

**Codebook / documentation** (needed to interpret `citl_information`
and `main_activity_code` — used to write the cleaning code correctly):
- Activity code mapping (power vs. industry grouping depends on this):
  `Translation of activity codes May 2019.xlsx`
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/Translation%20of%20activity%20codes%20May%202019.xlsx
  **This file is NOT in the data zip** — it must be fetched separately
  from this URL (or the codes transcribed from it) before step 6.
- EU ETS data viewer user manual (PDF) — may help interpret the
  `citl_information` metric hierarchy:
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/EEA_EUETS_data_viewer_user%20manual_June12.pdf
- Background note (PDF, Sept 2026, also included in the zip) —
  methodology, including the scope-correction estimates (item 3) and
  latest-year gap-filling (item 2.1b):
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/EU%20Emission%20Trading%20System%20data%20viewer%20Background%20note%20(september%202026).pdf
- Data quality note (PDF, also included in the zip) — known issues by
  country/year, worth a skim
  before trusting any single cell too far:
  https://sdi.eea.europa.eu/catalogue/api/records/9eee3fc9-3060-4e3f-bc8d-e22e7c3f6bda/attachments/ETC-CM%20EU-ETS%20data%20quality%20September%202026.pdf

**Zip contents** (confirmed): `ETS_Database_September_2026.xlsx` (the
data), `README.md` (general description only, no codebook), the
background note PDF, the data quality PDF, a metadata XML, and a JPG.
No CSV and no activity-code translation table.

**Columns in the data file** (confirmed, 90,325 rows):
`version`, `country_code`, `main_activity_code`, `active_installation`,
`citl_information`, `year`, `size`, `value`, `unit`.

This is a long/tidy table, not one with separate columns per metric.
What each column actually contains (confirmed by inspection in the
notebook, section 2):

- **`citl_information` holds the metric type** (not `size`). It is a
  numbered hierarchy of 19 labels. The ones relevant here:
  - `2. Verified emissions` and `2.1 EU-ETS Verified Emission` — same
    row count (11,151); likely duplicates of each other, **to be
    verified** before choosing one.
  - `2.1b EU-ETS Verified Emissions (expected, gap-filled for latest
    year)` and `2.1c Gap-fill increment (...)` — latest-year estimates.
  - `1.1 Freely allocated allowances` — total free allocation, with
    sub-items `1.1.1` (existing entities, Art. 10a(1)), `1.1.2` (new
    entrants reserve), `1.1.3` (Art. 10c modernisation of electricity
    generation), `1.1.4` (Swiss aviation).
  - `1.2 Correction to freely allocated allowances (not reflected in
    EUTL)` and `3. Estimate to reflect current ETS scope for allowances
    and emissions` — EEA adjustments, likely what the original plan
    expected to find as "pre-2013 estimated" flags.
  - Others not needed: `1.` total allocated, `1.3` auctioned,
    `2.2` Swiss aviation emissions, `4.x` surrendered units.
- **`size`** — always `"All sizes"`. Carries no information; ignore.
- **`active_installation`** — always `"all entities"`. Carries no
  information; there is no active-installation filter available.
- **`year`** — mixed type: 2005–2030 as integers (2026–2030 are
  allocation-only forward years) plus text rows
  `Total 1st/2nd/3rd/4th trading period (...)`.
- **`main_activity_code`** — text codes `10, 20–45, 50, 99` plus the
  aggregate codes `20-99` and `21-99`, which overlap the individual
  codes (summing everything would double count).
- **`country_code`** — 36 values: EU-27 plus `GB`, `IS`, `LI`, `NO`,
  `XI` (Northern Ireland), plus non-country entries `Innovation fund`,
  `Modernisation Fund`, `NER 300 auctions`, `RRF`.
- **`unit`** — `tonne of CO2 equ.` for emissions, `units` for
  allowances (1 allowance = 1 tCO2e, so these are comparable).
- **`version`** — always `82`.

## Cleaning Steps

1. ✅ **Done.** Pull the zip via `requests`, read the Excel workbook
   in memory, and load it into pandas.
2. ✅ **Done.** Inspect and print the unique values of `size`,
   `citl_information`, and `active_installation` (findings recorded
   above under "Columns in the data file").
3. Drop the uninformative columns `size`, `active_installation`, and
   `version` (each has a single value — assert this in code rather than
   assuming it, so a future data version with real categories fails
   loudly).
4. Clean `year`: drop the `Total ... trading period` text rows, convert
   to integer, and keep 2005–2025 only (drop forward allocation years
   2026–2030).
5. Verify whether `2. Verified emissions` and `2.1 EU-ETS Verified
   Emission` are identical (compare `value` on matching keys). Then
   filter `citl_information` to just:
   - verified emissions (one of the two labels above), and
   - `1.1 Freely allocated allowances`.
   Note how `2.1b` (gap-filled latest year) relates to 2025 — check
   whether the 2025 verified-emissions figure is incomplete without it,
   and decide whether to use `2.1b` for 2025 or flag 2025 as
   provisional. Leave out `1.2` corrections and `3.` scope estimates
   from the main series, but mention them as a caveat.
6. Fetch the activity-code translation table separately (URL above —
   it is not in the zip) and confirm what each code means, including
   `10`, `50`, `99`, and the aggregates `20-99` / `21-99`.
7. Drop the aggregate codes `20-99` and `21-99` to avoid double
   counting; work only from individual activity codes.
8. Drop aviation and maritime activity types (added to the scheme in
   2012 and 2024 respectively) — keep stationary installations only, so
   the time series isn't distorted by scope changes. Exact codes to be
   taken from the translation table in step 6 (do not assume `10` =
   aviation / `50` = maritime without checking).
9. Map the remaining `main_activity_code` values into two groups using
   the translation table:
   - **Power/combustion** (electricity and heat generation)
   - **Industry** (steel, cement, chemicals, refining, pulp & paper,
     etc.)
10. Drop non-country `country_code` entries (`Innovation fund`,
    `Modernisation Fund`, `NER 300 auctions`, `RRF`). Restrict to EU
    member states; decide explicitly on `GB` (left the EU ETS after
    2020, so it can't be in a balanced 2005–2025 panel), `XI`, and the
    EEA-EFTA states `IS`, `LI`, `NO`. Then keep countries reporting
    across the full 2005–2025 window to keep the panel balanced, and
    document which countries are dropped and why.
11. Pivot the filtered long table so each row is `country_code x
    main_activity_code x year`, with separate columns for verified
    emissions and free allocation (pivoting on `citl_information`,
    values from `value`).
12. Confirm units before combining: emissions should all be
    `tonne of CO2 equ.` and allowances all `units` (1 allowance = 1
    tCO2e). Assert this in code.
13. Aggregate emissions and free allocation by `year x group` (sum across
    countries), and also keep a `year x country x group` version for a
    robustness check.
14. Index emissions to a base year (2005 = 100) for each group, so the
    two series are comparable regardless of their different starting
    sizes.
15. Compute free allocation as a share of verified emissions, by group
    and year, to show the actual divergence in free-allowance exposure
    between the two sectors.
16. Check for missing years or country gaps and document them rather
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
  recent 1–2 years in the dataset may be provisional or incomplete. The
  file itself includes a gap-filled estimate for the latest year
  (`2.1b`), confirming 2025 is at least partly estimated.
- EEA publishes free-allocation corrections (`1.2`) and scope-change
  estimates (`3.`) that are not in the main series used here; the
  analysis uses the registry figures as reported.
- No price data is used here (EU ETS allowance price APIs are mostly
  paywalled or require a separate account), so this tests the effect of
  a *policy design change* (removal of free allocation), not the
  carbon price level itself.
