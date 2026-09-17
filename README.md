# Middleware Compliance Tracker

A single-file, offline web tool that adds first-found dates, SLA ageing and scan-coverage reporting to compliance scan exports that only carry the latest result.

Most compliance scan exports tell you what failed today. They do not tell you when it started failing, which makes SLA measurement impossible. This tool keeps that history itself, in a JSON file you control, and produces the report.

## Why

Compliance platforms commonly export a flat list of host, rule ID, rule name and result, with a last-scan timestamp and nothing else. Without a first-found date you cannot say how long a control has been failing, cannot measure remediation against an SLA, and cannot show an auditor that a control held over a period rather than on one day.

## What it does

- Reads a raw compliance export: CSV, TSV, Excel (.xlsx/.xlsm), or text pasted from the screen
- Joins it to a device inventory to carry owner, application and environment onto every finding
- Assigns and preserves first-found dates across runs, closes findings when the rule passes, and opens a new cycle when a closed finding fails again
- Ages every open finding against a configurable SLA and reports breaches, near-breaches and ageing buckets
- Reports scan coverage: inventory hosts missing from a run, inventory hosts never scanned in any run, and scanned hosts absent from the inventory
- Builds a standalone HTML report per run, including a basis-of-preparation section
- Exports a fixed-column CSV that pastes into an existing pivot-table workbook without breaking the pivot cache
- Supports correcting mistakes: full run rollback, or removal of individual findings and whole hosts, with a mandatory reason recorded in a change log

## Design constraints

**No server, no install, no dependencies.** One HTML file, opened from disk or a network share. It was written for an environment where a CDN request may be blocked by an egress proxy, so there are none. Excel files are read by parsing the ZIP container directly and inflating entries with the browser's own `DecompressionStream`, rather than loading a spreadsheet library.

**No data leaves the machine.** There is no network call of any kind in the page. Findings data stays in the browser tab and in the JSON file you save.

**The tracking file is the system of record, not browser storage.** `localStorage` is tied to one browser profile on one machine, is wiped by profile cleanup, and cannot be backed up or reviewed. A file on a shared drive can be. Where the File System Access API is available the tool writes back to the same file; otherwise it falls back to a download.

## Use

1. Open `middleware-compliance-tracker.html` in Chrome or Edge.
2. Open an existing tracking file, or start a new one.
3. Load the device inventory. It is stored inside the tracking file and only needs reloading when it changes.
4. Load the raw compliance export. Columns are matched automatically and can be corrected.
5. Set the scan date, operator and SLA, then load the run.
6. Save the tracking file, and build the report.

### Result values

Text results (`Pass`, `Failed`, `Non-Compliant`, `Warning`, and similar) are recognised directly. Numeric results are recognised too, where `0` is pass, `1` is warning and `2` is failed. Warnings are treated as open findings by default and aged like failures, since a check that did not pass is not evidence of compliance; they are labelled separately throughout and can be excluded per run, with the choice recorded in the run log and stated in the report.

### Dating rules

- A failure seen for the first time is dated from that run.
- A failure already on record keeps its original date however many times it reappears.
- A rule that moves to pass is closed and stamped with the run date.
- A closed finding that fails again opens a new cycle with a new date; the earlier cycle is kept in history.
- A host absent from an export is **not** treated as remediated. Its findings stay open and the host is reported under coverage.
- Findings present on the first tracked run have no observable start date. They are marked baseline, dated from the day tracking began, and labelled as such everywhere. Their true age is older than shown.

## Browser support

Chrome and Edge are the tested targets. Excel reading requires `DecompressionStream`, available in Chrome/Edge 103+ and Firefox 113+. Where it is unavailable, the tool says so and asks for CSV. Saving back to the same file requires the File System Access API; everywhere else the tool downloads the updated tracking file instead.

## Tracking file format

Plain JSON, human-readable, diffable:

```
{
  "schema": "middleware-compliance-tracker/2",
  "trackingStart": "YYYY-MM-DD",
  "slaDays": 30,
  "inventory": { "hosts": { ... }, "fields": { ... } },
  "runs":     [ { "id": 1, "scanDate": "...", "operator": "...", "undo": [ ... ] } ],
  "findings": { "host\u0000ruleid": { "firstFound": "...", "state": "open", ... } },
  "changes":  [ { "at": "...", "kind": "run-reverted", "reason": "..." } ]
}
```

Rollback works from the `undo` journal stored on each run, which records the prior state of every finding the run touched. Journals are kept for the ten most recent runs.

## Licence

MIT.

## Contributing

Issues and pull requests welcome. Please do not include real host names, rule output, inventory extracts or tracking files in issues — use synthetic data.
