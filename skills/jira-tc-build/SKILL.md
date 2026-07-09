---
name: jira-tc-build
description: >-
  Build a full test-case Excel workbook from a user story in the user's Jira Server (projects PROJ1, PROJ2, PROJ3 — any project key resolves dynamically).
  Takes a story key (e.g. PROJ1-5754, PROJ2-123, PROJ3-88), fetches summary +
  description + acceptance criteria via an
  authenticated stealth-browser REST session (read-only), designs exhaustive
  valid + negative test cases covering every acceptance-criteria line, and saves
  the workbook to <TC_OUTPUT_DIR> as
  "<KEY>_<Story title>.xlsx" in the exact column contract /jira-tc-upload consumes.
  Use when the user says "build test cases for PROJ1-xxxx", "jira tc build PROJ1-xxxx",
  "create the TC excel from this story", or gives a PROJ1 key and asks for test cases.
  Never writes to Jira — posting is /jira-tc-upload's job. Not for Jira Cloud.
---

# jira-tc-build — build test-case Excel from a Jira story

Fetches ONE user story from Jira Server (read-only), designs test cases covering EVERY acceptance-criteria line (valid + negative), and produces a formatted Excel workbook that `/jira-tc-upload` can post one row at a time. This skill takes NO public action: Jira access is GET-only through the user's own authenticated session; the only write is a local file.

## Known constants (shared with /jira-tc-upload — re-discover only if a call 400s)
- Base: `https://your-jira-host.example.com/jira` (REST base path `/jira/rest/api/2`)
- the user's projects (ids fetched live YYYY-MM-DD via `GET /rest/api/2/project/<KEY>`):
  | Key | Project id | Name |
  |---|---|---|
  | PROJ1 | 10001 | Example Project One |
  | PROJ2 | 10002 | Example Project Two |
  | PROJ3 | 10003 | Example Project Three |
- **Dynamic project resolution:** story key prefix = project key. Prefix in the table → use cached id. Any other prefix → resolve live: `GET /rest/api/2/project/<PREFIX>` → `.id` (authenticated page fetch, two-step async). 404 → unknown project, ask the user. On success append the new row to this table.
- Stealth profile: `spawn_browser {sandbox: false, user_data_dir: "~/.claude/stealth-profiles/jira"}` — persistent session, no login normally needed
- the user's username: `YOUR_JIRA_USERNAME`
- Output dir: `<TC_OUTPUT_DIR>` (mkdir -p before save; it may be recreated)
- Python for Excel: `<PYTHON_WITH_OPENPYXL>` (has openpyxl; plain `python3` on this Mac does NOT)

## Inputs
- **Story key** — REQUIRED. Normalize before use: `PROJ1-5754` stays; `PROJ1=5754` / `gr 5754` → `PROJ1-5754`; same for PROJ2 and PROJ3 prefixes; bare number with no prefix → ask which project (default PROJ1 only if she confirms). Anything not matching `^[A-Z]+-\d+$` after normalization → ask. Project id then resolves per the constants table (dynamic rule).
- Optional: extra business context the user pastes (compliance notes, channel list, card types). Fold into test design; never let it replace fetched AC.

## Hard rules
1. **Read-only against Jira.** GET requests only. Any create/update/link belongs to `/jira-tc-upload` with its own confirmation gate. Never POST/PUT/DELETE from this skill.
2. **Never handle her password.** Hard login form (`#os_username` / "You must log in") → ask the user to log in herself in the visible window, then continue.
3. **Exhaustive AC coverage** (the user's standing preference): every acceptance-criteria line and every business-requirement line maps to at least one Valid AND at least one Negative test case. No sampling. The AC Coverage sheet must prove it.
4. **Column contract is LOCKED** (what /jira-tc-upload parses — order and headers exactly):
   `TC ID | Test Case Title | Test Case Description | Preconditions | Test Data | Detailed Steps to Follow | Expected Result | Priority | Type`
5. **TC ID format:** `TC_<FEATURE>_01` sequential, where `<FEATURE>` is a short slug from the story (e.g. `FEAT`). /jira-tc-upload maps `TC_OTP_01` → `TC01` when posting.
6. **Filename:** `<KEY>_<Story title>.xlsx`, title sanitized for the filesystem: strip/replace `/ \ : * ? " < > |` with `-`, collapse whitespace, trim to ≤120 chars. Example: `PROJ1-5754_Credit cards payments FEAT.xlsx`.
7. **Existing file at target path → ask before overwrite.** Offer `_v2` suffix.
8. Priority column is for the workbook and test planning. /jira-tc-upload still posts everything as High (the user's standing Jira rule) — do not "fix" that here.

## Procedure

### 1. Fetch the story
1. `spawn_browser` with the persistent profile (constants above). Keep `instance_id`.
2. `navigate` to `<base>/browse/<KEY>`. Auth check: `!!document.querySelector('#create_link')`. Login form → rule 2.
3. GET the issue via the **two-step async pattern** (see Gotchas):
   `fetch('/jira/rest/api/2/issue/<KEY>?fields=summary,description,issuetype,status,priority', ...)` → stash on `window.__jtbFetch` → read in second `execute_script`. Expect 200.
4. If `description` lacks an "Acceptance Criteria" section, GET the full issue (no `fields` param) and scan custom fields for names containing "acceptance" (via `editmeta` or `/rest/api/2/field` cached list). Still nothing → show the user the description and ask her to paste/confirm the AC.
5. Sanity: issue should be a story/demand type, not a Test. If issuetype is an Xray Test → wrong key, ask.

### 2. Parse
- `summary` → story title (for filename + workbook header).
- `description` is Jira wiki markup: strip `h1.-h6.`, `*bold*`, `{code}`, `{quote}`, `{color}`; bullets (`*`, `#`, `-`) become AC line items.
- Enumerate AC lines AND business-requirement lines separately — business rules not restated in AC (e.g. "own card only") still get test cases.

### 3. Design test cases
For every AC/business line produce at minimum:
- **Happy path** (Valid) — the literal AC statement.
- **Boundary** (Valid) where numeric/amount/date logic exists — exact match, smallest-unit above/below.
- **Negative / defect guard** — the rule wrongly NOT applied (security-critical: bypass attempts, wrong/expired FEAT or token, unauthorized actor).
- **Cross-channel** cases when the story mentions channels/consistency — same behavior per channel + one inconsistency defect guard.
Assign Priority: `Critical` = security bypass/compliance breach; `High` = direct AC path; `Medium` = boundary/secondary/channel duplicate; `Low` = redundancy. Type = `Valid` or `Negative`.
Steps are numbered, one action per line, starting from login/channel entry. Test Data names concrete cards/amounts/users. Expected Result states observable outcome AND the pass/fail rule for defect guards.

### 4. Build the workbook
`<PYTHON_WITH_OPENPYXL>` + openpyxl. Three sheets:
1. **`<KEY> Test Cases`** — the 9 locked columns. Arial; header row bold white on `1F4E78`, frozen, autofilter; wrapped top-left cells; TC ID/Priority/Type centered. Fills: Type Valid `E2EFDA` / Negative `FCE4D6`; Priority Critical `C00000` white-bold / High `FFC7CE` / Medium `FFEB9C` / Low `D9D9D9`. Column widths ≈ 12/32/42/32/32/50/42/11/10.
2. **`AC Coverage`** — every AC + business line verbatim → covered-by TC IDs. A line with no TC = bug in this skill run; fix before saving.
3. **`Priority Legend`** — tier, meaning, count.
No formulas needed (static data) — if any are added, run recalc per xlsx-skill rules.

### 5. Save + report
1. `mkdir -p <TC_OUTPUT_DIR>`, save as rule 6 filename (rule 7 if exists).
2. Verify: reload workbook, assert sheet names, 9 headers exact, row count.
3. Report to the user: file path, TC count (valid/negative split), priority counts, the AC Coverage table inline, and any assumptions made (below-due behavior, channel list, etc.) flagged for her confirmation.
4. Ask before `close_instance` (she may chain into `/jira-tc-upload` with the same session).
5. Offer next step: "run `/jira-tc-upload` against this file to post them?" — do NOT start posting unprompted.

## Failure handling
- GET 401/403 → session died; re-auth via visible window (rule 2), retry once.
- GET 404 → key doesn't exist; show the key used, ask.
- Empty/one-line description → ask the user to paste the story text rather than inventing AC.
- openpyxl import error → verify interpreter is `<PYTHON_WITH_OPENPYXL>`, not `python3`.

## Gotchas
- **Async execute_script returns null** — tool serializes before promises resolve. Call 1: start async fn, write result to `window.__jtb<Name>`. Call 2: `JSON.stringify(window.__jtb<Name>)`. Poll if null.
- Sync reads: wrap in IIFE `(function(){ ... })()` — bare `return` throws SyntaxError.
- Jira wiki `{panel}` blocks often wrap AC on this server — strip the markers, keep the lines.
- Scratchpad may reset between turns — write generator scripts fresh; the xlsx is the durable artifact.
- Filename keeps spaces (the user's convention: `PROJ1-5754_Credit cards payments FEAT.xlsx`) — only forbidden filesystem chars are replaced.
