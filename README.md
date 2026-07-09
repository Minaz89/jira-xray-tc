# jira-xray-tc-pipeline

Two Claude Code skills that automate test-case work against a Jira Server + Xray instance:

1. **`/jira-tc-build <STORY-KEY>`** — read-only. Fetches a user story via Jira REST API v2 (through an authenticated stealth-browser session), extracts every acceptance-criteria and business-requirement line, designs exhaustive valid + negative test cases (happy path, boundaries, defect guards, cross-channel), and writes a formatted Excel workbook: 9-column test-case sheet + AC coverage matrix + priority legend.
2. **`/jira-tc-upload <STORY-KEY|xlsx-path>`** — takes that workbook and batch-creates one Xray Test issue per row: summary `TCnn | title`, steps + expected result in the Description field (Xray step grid intentionally empty), `Tests` link to the story, `Positive`/`Negative` label from the Type column, assignee + priority fixed. One explicit confirmation gate per batch, sequential POSTs with a 5-second pause between creates, continue-on-row-failure, server-side verification of every created issue.

The Excel file is the reviewable artifact between the two steps — build, review, then upload.

## Install

Copy `skills/jira-tc-build` and `skills/jira-tc-upload` into `~/.claude/skills/`.

## Configure

The skill files are sanitized. Replace these placeholders with your values:

| Placeholder | Meaning |
|---|---|
| `your-jira-host.example.com` | Your Jira Server host (REST base `/jira/rest/api/2`) |
| `YOUR_JIRA_USERNAME` | Your Jira username (assignee) |
| `PROJ1` / `PROJ2` / `PROJ3` + ids `10001/10002/10003` | Your project keys and ids (`GET /rest/api/2/project/<KEY>` — the skills also resolve unknown keys dynamically) |
| `<TC_OUTPUT_DIR>` | Local folder for generated workbooks |
| `<PYTHON_WITH_OPENPYXL>` | Python interpreter that has `openpyxl` installed |
| Xray Test issuetype id `11300` | Instance-specific — discover via `GET /rest/api/2/issuetype`, filter for the xpandit icon |

Authentication: the skills drive a persistent-profile stealth browser (nodriver/CDP MCP). First run: log in manually in the visible window; SSO session persists in the profile. The skills never handle passwords.

## Safety model

- `jira-tc-build` is strictly read-only against Jira (GET only).
- `jira-tc-upload` performs writes only after an explicit in-turn confirmation; batch mode shows the full pending list behind a single gate.
- 5 s pause between creates; 3 consecutive failures abort the batch.
