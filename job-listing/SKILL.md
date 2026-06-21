---
name: job-listing
description: Extract job listing data from a saved PDF or HTML page and append it to the 26-27 CJMS Grist postings tracker (Postings_Staging table). Use when the user drops a HigherEdJobs or similar listing for processing.
argument-hint: [path to file or list of files, e.g. "~/postings/baylor.txt"]
model: haiku
---

# Job Listing Extraction → CJMS (26-27)

Extract structured data from job listing(s) and append to Grist. Action requested: **$ARGUMENTS**

## Target (26-27 CJMS, Grist-only architecture)
- Doc: `cJNfNmdJjkwUQhrY1HoNyT` (26-27 CJMS source). Table: **`Postings_Staging`** (a hidden mirror of
  the live `Postings` table). **ALWAYS append to staging, NEVER to live `Postings`.** The user reviews /
  dedups staging, then promotes good rows into `Postings` themselves. Posting straight to `Postings`
  publishes unreviewed scrape output to the public grid — do not do it.
- Field names below are exact Grist column IDs; POST by these IDs.

## CRITICAL: No agents, no parallelization
**NEVER spawn subagents or parallelize.** Process files one at a time, sequentially. Two Bash calls per
file: read, then POST. That's it. Do NOT retry a failed curl. One confirmation line per file, then move on.

## Workflow — 2 Bash calls per file, sequential

**Call 1** — Read the file:
```bash
cat "/path/to/file.txt"
```

**Call 2** — Extract, POST, confirm:
```bash
GRIST_API_KEY=$(grep '^GRIST_API_KEY' ~/.config/grist.env | cut -d= -f2) && \
curl -s -X POST "https://docs.getgrist.com/api/docs/cJNfNmdJjkwUQhrY1HoNyT/tables/Postings_Staging/records" \
  -H "Authorization: Bearer $GRIST_API_KEY" -H "Content-Type: application/json" \
  -d '{"records": [{"fields": { ... }}]}'
```
Success response contains a `records` array with new ids. On an `error` object, report it; do NOT retry.

For dates, compute epoch inline: `$(date -d "YYYY-MM-DD" +%s)`. If no date, use empty string `""`. NEVER null.

## Example record
```json
{"records": [{"fields": {
  "Institution": "Eastern Illinois",
  "Title": "Assistant Professor of Public Relations",
  "Location": "Charleston, Illinois",
  "Link": "https://apply.interfolio.com/172166",
  "Area": ["L", "PR/Advertising", "Strat Comm"],
  "Requirements": ["L", "Cover Letter", "CV", "Transcripts", "Evidence of Teaching Effectiveness", "Writing Sample"],
  "Letters": "Names and contact only",
  "SpecialInstructions": "",
  "TenureTrack": "Yes",
  "Postdoc": "No",
  "Teaching": "Yes",
  "Load": "4/4",
  "Flags": ["L", "Is this a religious institution?"],
  "Ghost": "Not Known",
  "Timeline": "",
  "Source": "HigherEdJobs",
  "Contact": "Dr. Jane Doe, search-chair@example.edu",
  "DateAdded": 1759276800,
  "Other": "",
  "Deadline": 1759276800
}}]}
```

## Field schema (exact Grist column IDs)
Use empty string `""` for unknown free-text/choice/date values. NEVER null. NEVER true/false except where noted (there are NO boolean fields in this schema — flags are a ChoiceList now).

### Free text
| Field | Notes |
|-------|-------|
| Institution | Short name, e.g. "Baylor" not "Baylor University" — match existing style |
| Title | Exact title from the posting |
| Location | City, State, e.g. "Waco, Texas" |
| Link | Direct application URL |
| SpecialInstructions | Unusual/atypical application requirements ONLY (specific letter-content rules, format restrictions, visa-sponsorship notes, background-check details). NOT standard degree requirements, NOT which platform to apply on. Empty if none |
| Source | Where the listing was found. For this batch: `HigherEdJobs`. Use the actual source if the file says otherwise (`Chronicle`, `direct`, etc.) |
| Contact | Search chair / department name and/or email, whatever is given |
| Other | Role-specific responsibilities as listed (online/hybrid modality, program coordination, lab management, service leadership). NOT salary, NOT position length, NOT load/TT/degree reqs, NOT institutional boilerplate. Also used for uncertainty flags (see DateAdded). Empty if none |

### ChoiceList fields (arrays with "L" prefix) — `["L", "Value1", "Value2"]`, EXACT strings only

**Area** (pick ALL that apply):
`AI`, `Communication (Generalist)`, `Critical/Cultural Studies`, `Digital Media`, `Games Studies`, `Health Comm`, `History`, `Interpersonal/Group`, `Journalism`, `Mass Comm`, `Media Arts/Production`, `Media Studies`, `Org Comm`, `PR/Advertising`, `Performance Studies`, `Policy`, `Political Communication`, `Rhetoric/Speech`, `Sport Comm`, `Strat Comm`, `Visual Communication`
- There is NO `Other` choice in Area. Off-list areas → describe in the `Other` text column, do not invent an Area value.
- "Film"/"Film and Media" → `Media Studies` and/or `Media Arts/Production`; "speech"/"public speaking" → `Rhetoric/Speech`; "digital humanities" → `Digital Media`; "business/professional communication" → `Org Comm`. Err toward more.

**Requirements** (pick ALL that apply):
`Cover Letter`, `CV`, `Transcripts`, `Teaching Statement`, `Research Statement`, `Writing Sample`, `Evidence of Teaching Effectiveness`, `Diversity Statement`, `Sample Syllabi`, `Portfolio of Work`
- "Teaching philosophy" → `Teaching Statement`; "Resume" → `CV`; "syllabi"/"sample courses" → `Sample Syllabi`; "portfolio"/"creative work sample" → `Portfolio of Work`.
- Do NOT include "References"/"Letters of Recommendation" — the `Letters` field covers that. Anything off-list → `SpecialInstructions`.

**Flags** (pick ALL that apply) — these are FULL question strings; emit verbatim (the FlagsShort display formula keys on exact text):
`Is the institution a Small liberal arts college (SLAC)?`, `Is the institution an HBCU?`, `Is the institution a community college?`, `Does the position require production experience?`, `Does the position request relevant professional experience (i.e., journalism background)?`, `Does the job require debate/forensics coaching or supervision?`, `Is this a religious institution?`
- Empty list if none apply. Religious flag ONLY if the listing itself states affiliation — don't look it up.

### Single choice — exact strings, NEVER true/false
| Field | Valid values |
|-------|-------------|
| Letters | `Yes`, `Names and contact only`, `Letters at later stage` |
| TenureTrack | `Yes`, `No` |
| Postdoc | `Yes`, `No` |
| Teaching | `Yes`, `No` |
| Load | `2/2 or lower`, `3/3`, `4/4`, `5/5 or above` |
| Ghost | `Not Known` (always — not knowable from a listing) |
| Timeline | always `""` (not knowable from a listing) |

### Date fields — Unix epoch INTEGER (seconds), else `""`. NEVER null.
| Field | Rule |
|-------|------|
| Deadline | Use "review begins" date if no hard deadline. Set to ONE DAY BEFORE the advertised date (a deliberate one-day buffer). Verify with `date -d "YYYY-MM-DD" +%s` |
| DateAdded | **The listing's real "Posted" date from HigherEdJobs** → epoch. This drives sort-by-recency, so capture the true posting date, not today. **If the posting date is absent or ambiguous, set `DateAdded` to `""` AND add `posting date not found` to the `Other` field** so the user can fix it on review. Do NOT guess a date. Do NOT silently use today |

## Extraction guidelines
- Ambiguous value → best guess, note uncertainty in `Other`.
- Area: err toward more categories.
- Load: ONLY if explicitly stated ("4/4", "12 credit hours/semester", "3 courses/term"). Don't infer from title/type. "3/2" → "3/3". Else `""`.
- TenureTrack: "tenure-track" = `Yes`; "visiting"/"lecturer"/"NTT" = `No`.
- Letters = letters of recommendation, NOT cover letters (a cover letter is a Requirement). Required upfront = `Yes`; "references"/"names of references" or unmentioned = `Names and contact only`.
- Teaching: default `Yes` if Load is 4/4 or higher.
- Ghost always `Not Known`. Timeline always `""`.
- No salary anywhere (omit entirely).

## Response style
After appending, confirm concisely, e.g.:
- "Staged: Baylor — A/P Communication, Waco TX, deadline Mar 1, TT, 2/2, posted Feb 3"
- Flag any uncertain fields, and explicitly say if the posting date wasn't found.

Do NOT dump full JSON. Keep it short. (Promotion staging → `Postings` is a separate manual step the user does.)
