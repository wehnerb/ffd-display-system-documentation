# Incident Data Display — Data Analyst Questions

> These questions need to be answered before architecture or code decisions can be finalized for the incident data display worker. They are grouped by topic area.

---

## Data Format & Export

- How can data be exported from the incident documentation system — as a spreadsheet, pre-built graphics, or both?
- Can exports be pre-aggregated and ready to display, rather than raw incident records? The worker should not be performing statistical or grouping work.
- Can a single export include data for all stations in one file, rather than requiring a separate export per station?
- What unit identifiers exist in the data system, and can those be included in exports consistently?

---

## Data Structure

- Can data sheets use standardized header names — specifically `label`, `value`, `station`, and `unit` — or does the export format impose constraints on column naming?
- For line graph data specifically, can the analyst use `x` and `value` (or `period` and `value`) as headers, since the x-axis represents a sequence or time period rather than a discrete label?
- Will any data require multi-series charts (for example, multiple lines on a single graph comparing stations or units)? If so, can a `series` column be included?
- How will department-wide data (not specific to any station or unit) be distinguished from station-specific data in the export — blank station column, a special value like `all`, or something else?
- For incident type data, how many distinct values exist at the category, subcategory, and specific incident type levels? This affects whether pie charts or bar charts are readable at each aggregation level.
- Can exports include all three incident type hierarchy columns (category, subcategory, specific type) to allow the worker to filter or display at the appropriate level?
- Do performance benchmark times (baseline and benchmark values by incident type) exist in the data system, or would those need to be maintained manually in a config sheet?

---

## Time Period Filtering

- Can exports be filtered by time period, or will the full dataset always be exported and filtering happen elsewhere?
- What time periods are most operationally meaningful — last 30 days, month to date, year to date, last shift, or something else?
- Are rolling time periods (e.g., last 30 days recalculated daily) feasible, or only fixed periods (e.g., current month, current year)?
- How frequently can data be refreshed — near real-time, daily, or shift-based?
- Is automated export possible, or will it require manual updates each time?

---

## Pre-Built Graphics (if applicable)

- If pre-built graphics are provided instead of raw data, where would those files live? Google Drive is the current assumption.
- What file format would those graphics be in?
- Can the existing Google service account used by other department display workers be granted access to those files?

---

## Notes

- Station filtering will be handled via a `?station=` URL parameter. Unit filtering will use a separate `?units=` parameter (comma-separated for multi-unit stations). The export format should support both.
- The worker will filter to rows matching the requested station or unit. Rows with a department-wide designation will show on all displays regardless of station.
- Data for turnout times and similar metrics should use unit identifiers rather than station identifiers, so that if a unit relocates to a different station, only the display URL needs updating — not the data or configuration.
