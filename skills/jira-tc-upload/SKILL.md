---
name: jira-tc-upload
description: >-
  Upload Xray test cases to the user's Jira Server (projects PROJ1, PROJ2,
  PROJ3 — project id resolves dynamically from the story key prefix), linked to
  their user story and labeled Positive/Negative, following her fixed
  convention. Two modes: batch (read all rows from a /jira-tc-build Excel in
  <TC_OUTPUT_DIR> and post each as an Xray Test) or
  single (one pasted table row/screenshot). Use when the user says "upload the test
  cases", "post the TCs to jira", "jira tc upload PROJ1-xxxx", "create a testcase
  in xray", or "add a TC to PROJ1-xxxx / PROJ2-xxxx / PROJ3-xxxx". Creates via Jira
  REST API v2 through an authenticated stealth-browser session; never handles
  her password. Not for Jira Cloud.
---

# jira-tc-upload — upload Xray test cases

Creates Xray Test issues in Jira Server, linked to a user story, matching the user's exact convention. Batch mode (default): reads the Excel `/jira-tc-build` produced and posts every row. Single mode: one pasted row. **Method: REST API v2 called via `execute_script` fetch() from an authenticated stealth-browser page.** Do NOT drive the Create dialog UI — the AUI issue-type dropdown is unreliable under automation (suggestion ids regenerate per keystroke; synthetic selection intermittently loads the wrong form; failed 2 of 3 live runs). REST worked first-shot with server-side verification.

## Known constants (this server — discovered YYYY-MM-DD via REST; re-discover only if a call 400s)
- Base: `https://your-jira-host.example.com/jira` (REST base path `/jira/rest/api/2`)
- the user's projects (ids fetched live YYYY-MM-DD via `GET /rest/api/2/project/<KEY>`):
  | Key | Project id | Name |
  |---|---|---|
  | PROJ1 | 10001 | Example Project One |
  | PROJ2 | 10002 | Example Project Two |
  | PROJ3 | 10003 | Example Project Three |
- **Dynamic project resolution:** project = story key prefix (`PROJ2-123` → PROJ2). Prefix in table → cached id. Other prefix → `GET /rest/api/2/project/<PREFIX>` → `.id`; 404 → ask the user; success → append row here.
- Xray Test issuetype 11300 + `Tests` link type verified on **PROJ1 only**. First create in PROJ2/PROJ3: verify via `GET /rest/api/2/issue/createmeta?projectKeys=<KEY>&expand=projects.issuetypes` that issuetype 11300 exists in that project's scheme BEFORE the checkpoint; if absent, discover the project's Xray Test id (xpandit iconUrl) and note it in the table.
- **Xray Test issuetype id: 11300** ("Represents a Test", icon `com.xpandit.plugins.xray/images/test.png`)
- Generic Test issuetype id: **10502** — NEVER use (icon `viewavatar?avatarId=10003`)
- Issue link type: name **`Tests`** (outward `tests`, inward `tested by`)
- the user's username: **YOUR_JIRA_USERNAME**

## Inputs (gather before starting)

**Batch mode (default when she names a story key or an Excel file):**
- Excel path, OR story key → resolve to `<TC_OUTPUT_DIR>/<KEY>_*.xlsx` (glob; multiple matches → ask which; none → suggest running `/jira-tc-build` first).
- Story key = filename prefix before first `_` (validate `^[A-Z]+-\d+$`). Prefix determines project id (constants table).
- Rows read with `<PYTHON_WITH_OPENPYXL>` + openpyxl from the `<KEY> Test Cases` sheet (fallback: first sheet). Expected 9-column contract: `TC ID | Test Case Title | Test Case Description | Preconditions | Test Data | Detailed Steps to Follow | Expected Result | Priority | Type`. Headers mismatch → show diff, ask before proceeding.

**Single mode:** a pasted table row (same columns, Priority/Type optional) or a screenshot, plus story key. Story key missing → ask.

**Both modes:**
- **TC number** — sequential `TC01`, `TC02`, ... derived from the TC ID (`TC_OTP_01` → `TC01`).
- Priority: NOT an input — always **High** (the user's standing rule, YYYY-MM-DD). Workbook Priority column is planning-only; ignore for the payload. Only deviate if she explicitly says so.
- **Label** — from the Type column: `Valid` → `Positive`, `Negative` → `Negative`. Missing/other Type → ask (do not guess).

## Hard rules (the user's convention — never deviate)
1. **Issue type:** Xray Test = **issuetype id 11300**. By id only — never by dropdown position or name matching.
2. **Summary:** `TC<nn> | <title>`.
3. **Description field** = description text + `Steps to follow` numbered list + `Expected result` paragraph (rule updated YYYY-MM-DD: Expected result INCLUDED). No precondition / test data / external ref unless the user says otherwise.
4. **Xray Manual Test Steps grid stays EMPTY** — REST create never touches it; do not add steps post-create.
5. **Link:** `Tests` link type, **outward** from the Test to the story (`tests → PROJ1-xxxx`), passed in the create payload's `update.issuelinks`.
6. **Assignee = the user** (`YOUR_JIRA_USERNAME`), **Priority = High always** (standing rule).
7. **Public-action gate:** get explicit "yes create" in-turn BEFORE any POST. Single mode: show the full payload (summary, description verbatim, priority, label, link). Batch mode: ONE gate for the whole batch — show a table of every pending create (`TC<nn> | title | label | link → <STORY>`) plus one full sample payload; explicit "yes create" covers the listed batch and nothing else. New/changed rows after the gate → re-gate.
8. **Labels:** every Test gets exactly one of `Positive` | `Negative` (mapped from Type: Valid→Positive). Payload: `"labels": ["Positive"]`. Never invent other labels.

## Procedure
0. **Batch only (local, before any Jira call):** resolve the Excel (Inputs), read all rows, derive story key from filename, map each row → `{tcNum, summary, description, label}`. Show row count + story key. Empty sheet or unparseable rows → stop, ask.
1. `spawn_browser` `{sandbox: false, user_data_dir: "~/.claude/stealth-profiles/jira"}` — persistent profile keeps her session (VERIFIED YYYY-MM-DD: fresh spawn → authenticated, REST /myself 200, no login).
2. `navigate` to `<base>/browse/<STORY>`. If a hard login form appears (`#os_username` / "You must log in"), ask the user to log in herself in the visible window — never handle her bank password. Confirm authenticated: `!!document.querySelector('#create_link')`.
3. Build the payload:
   ```json
   {
     "fields": {
       "project":   {"id": "<PROJECT_ID from story key prefix — constants table>"},
       "issuetype": {"id": "11300"},
       "summary":   "TC<nn> | <title>",
       "description": "<description>\n\nSteps to follow\n1. ...\n\nExpected result\n<expected>",
       "assignee":  {"name": "YOUR_JIRA_USERNAME"},
       "priority":  {"name": "High"},
       "labels":    ["<Positive|Negative — rule 8>"]
     },
     "update": {
       "issuelinks": [{"add": {"type": {"name": "Tests"}, "outwardIssue": {"key": "<STORY>"}}}]
     }
   }
   ```
4. **CHECKPOINT** (rule 7): single → present payload; batch → present the pending-creates table + one sample payload. Wait for explicit yes.
5. POST via the **two-step async pattern** (see Gotchas): `fetch('/jira/rest/api/2/issue', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload)})` → stash `{status, body}` on `window.__jtcCreate` → read it in a second `execute_script`. Expect **201** with `{key: "PROJ1-xxxx"}`.
   Batch: POST rows SEQUENTIALLY (one at a time, verify 201 before next — no Promise.all; parallel creates risk rate-limit and out-of-order TC numbers). **Pause 5 seconds between each create** (server courtesy — the user's rule, YYYY-MM-DD): in-page `await new Promise(r => setTimeout(r, 5000))` before each POST after the first, or wait between execute_script calls. Never batch faster even if the server looks fine. A row failure does NOT stop the batch: record `{tcId, status, error}`, continue, report all failures at the end. 3 consecutive failures → stop, report, ask.
6. Verify server-side: `GET /rest/api/2/issue/<newKey>?fields=summary,issuetype,assignee,priority,labels,issuelinks,description` → assert issuetype id 11300, assignee, priority, label present (rule 8), `tests → <STORY>` in issuelinks, description contains "Expected result". Batch: verify all created keys in one loop after posting.
7. Report: table of `TC<nn> | new key | URL | label | verified ✓/✗` (single row for single mode) + failed rows with reasons. Ask before `close_instance` (she may keep working).

If discovery is ever needed again (constants drift, new project): `GET /issuetype` (filter name "Test" + xpandit iconUrl), `GET /project/<KEY>`, `GET /issueLinkType`, `GET /myself` — all via authenticated page fetch.

## Failure handling
- POST 400 → body names the bad field; fix payload, re-checkpoint if content changed.
- POST 401/403 → session died; re-run step 2 (the user logs in), retry once.
- Constants 404 → re-run discovery, update this file's constants.
- Do NOT fall back to driving the Create dialog UI without asking the user first.

## Gotchas
- **Async execute_script returns null** — the tool serializes before promises resolve. Pattern: call 1 starts the async fn and writes result to `window.__<name>`; call 2 returns `JSON.stringify(window.__<name>)`. Poll if still null.
- Sync reads: wrap in IIFE `(function(){ ... })()` — bare `return` throws SyntaxError.
- Screenshots auto-save to file — Read the returned path. Prefer REST GET verification over screenshots.
- Description is Jira wiki text; plain lines + numbered list render fine.
- The UI Create dialog (legacy path, avoid): two "Test" entries whose order flips by recency; icon `xpandit` = real one; hidden `select` vs visible combobox desync; `mousedown+mouseup+click` needed on `#create-issue-submit`; issue-type change resets the form. Kept here only as forensics for why REST is the method.
