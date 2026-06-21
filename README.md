# A Discipline-Agnostic Academic Job Tracker

## 1. What this is

Hello. This provides a template for a public academic job-postings tracker, an intake form, and a "fork-your-own" personal
application tracker — plus an optional placement-data resource — built entirely on [Grist](https://www.getgrist.com/). This guide is for someone in another discipline who wants to stand up the same thing for their field.

> **Just want to use the Communication tracker?** You don't need any of this. Open the [template](https://docs.getgrist.com/6GWSuK4pPXE4/26-27-comm-job-market-spreadsheet-your-version) and hit **Duplicate Document** to get your own copy. The rest of this guide is for porting the whole system to another field.

I don't know about your discipline, but mine (Communication, nominally) is not very good at centralising and standardising job posting data. Each of us must toil across different job boards, or, forbid, the job wiki, and track each application individually. So divided, we beg. 

These templates are designed to address these issues, aggregating postings and creating a tracker from said postings.  
 
You don't need to self-host anything. Everything runs on getgrist.com's free tier, and the only external dependency (the sync widget) can now live inside the Grist document itself. What you get:

Two pieces, separable:

- **Postings tracker** — a public, read-only Grist doc holding a Postings table (one row per job ad), fed by a published intake form and/or a scraping skill. Anyone can read it. Anyone can **Duplicate Document** a pre-built *template* into their own free Grist account and get a personal application tracker — a sync widget pulls the public feed into their copy, and they layer their own private status/notes on top.
 Examples of what this looks like (for Communication) are [here for the aggregated postings spreadsheet](https://docs.getgrist.com/cJNfNmdJjkwU/26-27-Comm-Job-Market-Spreadsheet), and [here for the postings tracker](https://docs.getgrist.com/6GWSuK4pPXE4/26-27-comm-job-market-spreadsheet-your-version).

- **Placement data** — a separate public resource recording *who was actually hired* (the other end of the same timeline), fed by a public form and a private outreach campaign to department/search chairs. 

The architecture can be the same for most humanities/social science disciplines, I think; what changes is the vocabulary/verbiage (areas of study, application requirements, etc)

**Files in this repo**
- `README.md` — this guide
- `LICENSE` — AGPLv3 (full text)
- `postings-sync-widget.html` — the Grist sync widget
- `hej-clipper.js` — the HigherEdJobs capture bookmarklet
- `job-listing/SKILL.md` — the Claude Code capture skill

---

## 2. Architecture at a glance

```
PUBLIC (you own)                          PER-USER (they own)
┌────────────────────────┐                ┌────────────────────────┐
│ SOURCE doc             │   Duplicate    │ TEMPLATE fork          │
│ • Postings (feed)      │ ─────────────► │ • Postings (mirror)    │
│ • Intake Form (public) │                │ • Tracking (their own) │
│ • Calendar view        │   sync widget  │ • Calendar view        │
│                        │ ◄── fetches ── │ • inline sync widget   │
└────────────────────────┘   feed (anon)  └────────────────────────┘
```

- **SOURCE doc** — the canonical public feed. Public read-only. Holds the real postings, the published intake form, a calendar view.
- **TEMPLATE doc** — a normal Grist doc set up as the fork target (not a Grist *Template*-type doc; the forkable behavior comes from the "Special rules for templates" access rule, see §3 step 4): a Postings *mirror* table, a separate Ref-linked Tracking table the forker owns, a calendar, and the sync widget already installed. The forker hits **Duplicate Document** and is ready to go with zero setup.
- **Sync widget** — a single self-contained HTML file that, on open + a manual Refresh, fetches the public SOURCE feed anonymously and idempotently upserts rows into the fork's local mirror (`BulkAddRecord`/`BulkUpdateRecord` split on a key field; non-destructive, never deletes). It now lives **inside the doc** via a builder widget — forkers host nothing.

**Why two docs, not one.** Merging them self-syncs (the source feeding itself), and breaks a few other things, like making the form write locally per fork. The split is load-bearing, in other words.

---

## 3. Fork checklist (standing up your own SOURCE + TEMPLATE)

The fastest path is to **copy the existing Communication docs** and re-vocabulary them (§5), rather than rebuild from scratch.

1. **Get a free Grist account** at getgrist.com (or self-host Grist core; the guide assumes SaaS).
2. **Copy the SOURCE doc** into your own workspace. Clear the sample rows.
3. **Copy the TEMPLATE doc** into your own workspace.
4. **Re-apply "Special rules for templates"** on the TEMPLATE (Access Rules → Special Rules). This is the toggle that lets non-owners copy/download the doc for template use. ⚠️ **It does NOT survive a copy** — you must re-apply it on every distributed/forkable doc, every time you re-ship.
5. **Re-apply the hide-history access rule** on every public/distributed doc: add a dummy table and an access rule `user.Access != OWNER → deny -CRUD`, then hide that table's page from nav. Grist's Document History is otherwise visible to all viewers (including anonymous) and **leaks the editor's email**. The rule *hides* history, it does not erase it.
6. **Point the sync widget at your SOURCE doc.** Edit the config constant at the top of `postings-sync-widget.html` (the source doc ID + table name), then paste the whole file into an in-doc builder widget on the TEMPLATE's grid page: add a Custom section (Data = your mirror table), pick the `@berhalak/custom-widget-builder` gallery widget, paste the HTML verbatim, set **Access = Full**. The pasted code *and* the Full-access grant survive a non-owner Duplicate Document — forkers do zero widget setup. (GitHub Pages hosting of the widget is now only a fallback.)
7. **Publish the intake form** (SOURCE doc → the form view → Publish). Grab the published URL **from the UI** — published-form URLs are not API-derivable
   (the form's `linkId` is not the URL key).
8. **Make the SOURCE doc public read-only** so the widget can fetch it anonymously (see the CORS note in §6).
9. **Add a Document Tour** to the TEMPLATE (optional but recommended): a `GristDocTour` table drives an onboarding walkthrough that anchors to UI cells. You can see the example in both Communication Spreadsheets. You certainly don't need to copy my tracker tour. 
See the gotchas in §6 for the cell-anchor mechanics.

When you ship an update, re-do steps 4–5 on the shipped copy (the rules don't survive copies), but keep the doc ID stable so existing template links/tour fragments don't break.

---

## 4. Feeding the feed (intake automation)

Postings automatically populate via the form. I don't know where your discipline posts jobs, but for us, it's mainly Higher Ed Jobs and a few other sundry job sites. The hardest part of this whole thing is probably going to be getting people to submit jobs via the form. What I'd initially hoped was that everyone would automatically add jobs via the form as they updated the wiki (I posted the spreadsheet on the wiki), or as they posted it to HEJ or wherever, but this didn't happen. In Communication, at least, in the first year I didn't really reach that level of awareness from search committees. Maybe next year. So basically I ended up spending a few hours each Sunday manually entering in job postings from saved searches across HEJ, ICA, etc. 

After a few months of this I decided to lever the miracles of machine learning to save myself some time. What I did was set up a pipeline by which an LLM model read postings and automatically posted updates via API to a staging document that I would then check over/correct and manually promote them to the table. This cut down a job of 2-3 hours to under half an hour.

So what I did was create a `Postings_Staging` table — a hidden mirror of `Postings` in the SOURCE doc, made **owner-only** with an access rule (`user.Access != OWNER → deny -CRUD`) and hidden from public nav. All automated capture writes here. 

I built (or rather, instructed Claude to build) two capture tools that chain to feed staging. You can use this chain, or wire your own, depending on how complicated you want to make it and how many tokens you have to burn. I'm sure there's some openclaw nightmare you could set up. 

1. **A browser bookmarklet** (`hej-clipper.js`) — run it on a HigherEdJobs posting page; it reads the page's `JobPosting` structured data (title, institution, location, posted date, apply URL, employment type, full description) and downloads a `.txt`. It's **board-specific** — the selectors target HigherEdJobs, so you'd have to re-target these structured data fields for different boards. To install it, make a new browser bookmark and paste the entire contents of `hej-clipper.js` as the bookmark's URL (it's a `javascript:` bookmarklet); then click it while viewing a posting. 

So basically I would go through the saved search and manually open + bookmarklet a bunch of postings. This method also has the fringe benefit of preserving the job posting text as .txt for when/if I wanted to reference it later. It also cuts down on a bunch of the html cruft to save tokens for the:

2. **Claude Code skill** (`job-listing`) — takes that saved `.txt` (or any PDF / HTML / text listing), extracts the structured fields, and POSTs them to `Postings_Staging` over the Grist REST API. Encoding rules that bite: ChoiceList values as `["L", …]`, dates as epoch seconds, empty string never `null`. The skill needs a Grist API key to write: create one in Grist (Profile Settings → API Key) and put it where the skill reads it — it expects `GRIST_API_KEY=...` in `~/.config/grist.env`. 

I had pretty good success running this on Haiku, although obviously Sonnet was more accurate. This can burn quite a few tokens, depending on how many postings you have, if you're on a lower plan, you may want to space it out a little over the weekend or something. You will need to change some of the language in the skill to adjust it to your discipline's expectations; the job-listing skill is wired specifically to HEJ and Communication's sub-disciplinary schema, so whatever flags you choose to introduce (first year comp for English, for example) should be reflected in the skill. 

---

## 5. The "change these" map (what's discipline-specific; what you need to do)

Almost everything ports unchanged. The parts that are Communication-specific and that **you must adjust the vocabulary** for your field:

| Surface | What it is | Comm value (example) | Change to |
|---|---|---|---|
| `Area` | areas of study on the intake form dropdown | 21 Comm subfields | your field's subfields, standard expectations |
| `Flags` | institution-type flags | full-question multi-select (SLAC? HBCU? CC? production? religious? debate?) | your field's relevant institution types, if any |
| `Requirements` | application materials requested | Cover/CV/Transcripts/Research/Teaching/Writing/Diversity statements, Syllabi, Portfolio, Teaching evidence | your field's materials |
| `Other` | free-text escape hatch for off-list Area | — | keep as-is |
| terminal-degree types (placement only) | degree kinds your field hires from | MFA / JD / PhD (Comm is unusually broad) | your field's spread |

**Gotcha that bites here:** the grid/card uses display-only formula columns (`FlagsShort`, `RequirementsShort`) that map the verbose choice text to short tags. **These formulas are keyed on the EXACT choice text** — editing a `Flags`/`Requirements` wording silently breaks the mapping (unmapped values pass straight through). Re-map the formula whenever you change the wording.

Everything else — the two-axis tracking model, the calendar, the widget, the access rules, the form mechanics — is field-agnostic.

**The tracking model** (in the fork, owned by the forker, so they can re-colour or re-label freely):
- `Status` — pipeline: Interested / Applying / Applied / **Skip**
  (`Status = Skip` is the hide mechanism: a pinned grid filter excludes Skip).
- `Result` — response: Materials requested / Phone interview / Campus visit /
  Offer / Rejected / Declined.
- `Letters_Sent` — To request / Requested / Submitted.
- `Notes` — free text.

It is **flat** (one row per posting), deliberately *not* the official Grist Job Application Tracker's relational milestone-log (that's for solo data entry; this
is a shared feed).

---

## 6. Gotchas appendix (postings)

- **`AddColumn` without `isFormula: False` silently creates a FORMULA column** (non-editable). Fix with `ModifyColumn isFormula: False`.
- **`TODAY() + 7` errors.** Grist formulas are real Python, so date + int is a type error. Use `($Deadline - TODAY()).days`. Deadline conditional formatting
  uses three hidden `gristHelper_` rule columns (red ≤7d incl. overdue / amber ≤21 / green beyond).
- **ChoiceList values must be `['L', ...]`-encoded** when writing via the API.
- **CORS:** getgrist.com returns `access-control-allow-origin: *`, so an anonymous cross-origin GET works in-browser. But `Authorization` is **not** in the allowed headers — an *authed* cross-origin fetch is blocked. Hence the source must be public-read and the widget fetches anonymously.
- **Rate limits** (free tier): 5,000 calls/doc/day, 5 req/sec/doc, 10 concurrent/doc. These kill *background polling*, not direct fetch. The widget uses poll-on-open + manual Refresh + 429 backoff — never background-poll.
- **Non-owner "no write access" on sync is expected**, not a widget bug: the forker is looking at the *shared template*, not their own copy. Guide line: "if you see no-write-access, Duplicate Document first." (The widget's own errors are only source-fetch / rate-limit.)
- **Document Tour anchors:** `GristDocTour` columns are read by colId (Title/Body/Placement/Location + Link_URL/Link_Text/Link_Icon). `Location_Cell` = the URL tail `/p/<view>#a1.s<section>.r<row>.c<colRef>`; `Location` = formula `SELF_HYPERLINK() + $Location_Cell` (re-resolves per copy). Grab anchors with **Ctrl+Shift+A**. Custom widgets are not anchorable (no cell) — anchor a nearby cell. `Link_Icon` must be an exact Grist `IconList` name. Launch the tour directly via the `#repeat-doc-tour` URL fragment.

---

## 7. The placement-data extension

The other end of the same timeline: the postings tracker captures the ad, this captures who actually got hired. Same entities, later lifecycle stage. Modelled on APDA (philosophy); comparable exemplars are APSA placement data, the Lawsky entry-level hiring report (law), and PhilJobs appointments.

### Building the send list

The real work here is logistical, not technical: you basically need to assemble and email a list of several hundred people asking them who they actually hired.
The "Contact" email field should get you some of the way, but not everyone will include that in a job posting. 
For this, basically, I pointed an Opus model at the problem and asked it to derive a research plan to fill in the gaps, giving them a spreadsheet of departments that had posted a listing. It then launched several smaller agents to pull a contact name for the department head from public-facing directories for the ~200 or so listings that were missing a point of contact. Where there was a named search chair without email, I instructed it to search for and use that email; I defaulted back to a listed department chair where there wasn't a search committee name. 
There was a fair amount of trial and error in these initial few runs and I think it only really got about ~90% of them, and I did the rest manually. And this is the first year I've done this so it remains to be seen how many entries I'll actually get, but I'm optimistic. 

From there I had one big spreadsheet that I used to feed a mail-merge list. I wrote an email introducing myself and the project, and asked them to submit who they hired via the public form. I tracked responses in the same spreadsheet and manually updated it as and when responses came through. I decided to initially keep the details collected pretty sparse for the first run, just public facts about the hire: name, terminal-degree year and school, last position (Postdoc/VAP/Lecturer/PhD Institution/Other, with option to give details about Other), optionally a link to the hire's own public CV/professional page (just a pointer to public info). I kept the form to structured public fields initially; maybe in the future I'd ask stuff about applicant numbers, number of publications, etc, but I figured that would chill responses too much and wouldn't be consistently on-hand in the same way (and can probably be backwards derived)

### The two docs

The placement data lives in two separate Grist docs, and the split matters for the same reason it does on the postings side: one is public, one is private, and they never touch in any published structure.

**The public doc** is really two linked tables (there's a throwaway third one, but that's just the dummy table for the hide-history rule from §3, so ignore it):

- **Institutions** is the canonical list of schools, one row each. I seed it from the departments that posted a listing, so it starts life as basically the deduped institution column of the postings spreadsheet. It then does two jobs at once. First, it's what populates the form: the form's institution question is a *reference* into this table, so when someone goes to report a hire they pick their school from a dropdown of known institutions rather than free-typing it. Second, it's the public-facing display; each institution row surfaces its most recent reported hire (the name, terminal-degree year and school, last position, and CV link) by pointing at the latest matching response.
- **Placements** is the raw response stream, one row per form submission, each one linked back to an Institution. If a respondent's school isn't in the dropdown yet, they drop it into a write-in field ("hiring institution, if not listed above"), and I fold that into Institutions afterwards.

So the loop is: seed Institutions from who posted, the form offers those as a dropdown, responses land in Placements pointing back at an institution, and Institutions surfaces the latest placement per school for anyone reading. Seeding the dropdown off the postings is also what keeps the two halves of the project (postings and placements) lined up on institution name without my maintaining a separate master list.

**The private doc** is, honestly, just the job-postings spreadsheet again, plus the scratch table where I did the contact lookup. The main table is the postings carrying a chair/email column; the second table (the dedupe scratch) is where the agent-driven lookup added the dept-chair name and email, the search-chair name and email, and a fallback contact, and where I deduped down to one contact per department. That deduped table is the mail-merge send list. Because the contacts only ever live here, the public doc stays clean, with no emails anywhere on the public surface. The only thing the two docs share is institution name, which is all you need to do the "don't re-email the people who already replied" diff privately at merge time. It isn't a grand safety apparatus, just the obvious consequence of keeping the contact list in its own doc.

---

## 8. Reference: the Communication instances

(For those who want to look at a live example. These are the Comm docs — copy their structure)

- Postings SOURCE doc (public feed): [26-27 Comm Job Market Spreadsheet](https://docs.getgrist.com/cJNfNmdJjkwU/26-27-Comm-Job-Market-Spreadsheet)
- Postings TEMPLATE (fork target): [26-27 comm job market spreadsheet (your version)](https://docs.getgrist.com/6GWSuK4pPXE4/26-27-comm-job-market-spreadsheet-your-version)
- Placement responses (public): [25-26 comm placement data](https://docs.getgrist.com/qe3Xp8D6uKcA/25-26-comm-placement-data)
- Sync widget (canonical HTML / fallback host): [srmce.github.io/cjms-widget/postings-sync-widget.html](https://srmce.github.io/cjms-widget/postings-sync-widget.html) (repo `srmce/cjms-widget`)
- Capture bookmarklet (HigherEdJobs clipper): `hej-clipper.js` (bundled in this repo)
- Capture skill (Claude Code): `job-listing/SKILL.md` (bundled in this repo; drop into your `.claude/skills/`)

> **License.** The code in this repo (the sync widget, `hej-clipper.js`, and the
> `job-listing` skill) is released under the **GNU Affero General Public License
> v3.0** ([AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html); see `LICENSE`).
> This guide is released under **[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)**.
> Fork it and adapt it to your field, but keep it open: anything you build on it,
> including anything you run as a hosted service, stays under the same terms.
