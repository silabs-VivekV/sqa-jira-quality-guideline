---
name: sqa-jira-quality-guideline
description: >-
  SQA Jira bug ticket quality checklist and minimal-input auto-fill workflow.
  Use only when the user query contains SQA_JIRA (case-sensitive). Always show a
  draft first; create in Jira only after the engineer replies with FILE_JIRA.
#disable-model-invocation: true
---

# SQA Jira Bug Quality Guideline

## When to use

Activate this skill **only** when the user query contains `SQA_JIRA` (case-sensitive) or the engineer attaches `/sqa-jira-quality-guideline`.

**Filing gate:** always show a draft first. Call Jira create/update **only** after the engineer replies with **`FILE_JIRA`** (see Draft approval before create).

## Cursor disclaimer (mandatory)

Always prepend this exact disclaimer line at the **top** of:

1. Every Jira bug **Description** draft shown to the user, and the Description body on `FILE_JIRA` create/update
2. Every Jira **triage comment** body (any `jira_add_comment` / comment update that contains Cursor triage analysis, RCA notes, or a proposed PR/fix)

```markdown
> **Disclaimer:** This triage and proposed PR is Cursor generated. Recommend manual review before accepting this fix.
```

Rules:

- Prepend as the **first** line(s) before any other markdown (`## Summary`, repro steps, etc.).
- Do not omit, reword, or relocate the disclaimer below other content.
- Include it in the in-chat Description draft **and** in the Description passed to `jira_create_issue` / `jira_update_issue`.
- If editing an existing triage comment, keep this disclaimer as the first line of the body.
- Source of truth for the text: `field-defaults.yaml` → `triage_comment_disclaimer` (same string for Description and comments).

## Why this matters

Recent triage runs show repeated delays due to incomplete evidence in Jira tickets (especially screenshot-only logs, missing repro commands, and missing baseline comparisons). Better ticket quality directly improves:

- RCA confidence
- PR readiness
- time to closure

---

## Automated ticket fill (minimal user input)

Goal: create a triage-ready Jira bug with **very few user inputs**. The agent fills the rest from (in order):

1. **User override** (explicit in the `SQA_JIRA` message)
2. **Parsed Summary** (structured format below)
3. **Atlassian Cloud test case** (mandatory 3rd Summary bracket — `SW_SQA_TC-*` / browse URL)
4. **Skill defaults** in `field-defaults.yaml` (same folder as this skill)
5. **Ask once** only if still missing and required to create the issue

**Hard gate:** Always show a draft and wait for explicit user approval before any `jira_create_issue` / create-side update. Phrases like “create a JIRA” or a Summary alone are **not** approval — see **Draft approval before create**.

### Phase 1 — start with Summary*

User must provide **Summary\*** in this preferred format (use verbatim):

`[<product-or-device>][Affects Version][<Atlassian-TC-key>] <observed-behavior> instead of <expected-behavior> when <condition>`

The **third bracket is an Atlassian Cloud test-case issue key** (not a free-form feature name). Prefer the key alone; a full browse URL is also accepted and must be normalized to the issue key.

Examples:

```text
[SiWx917-Soc][26Q2-GA][SW_SQA_TC-11261] DUT returns empty scan list instead of SSIDs when associated and BG scan is enabled
[SiWx917-NCP][3.5.0][SW_SQA_TC-11261] Association fails with 0x10003 instead of success when AP uses WPA3-SAE only
[SiWx917-RCP][26Q1][SW_SQA_TC-11261] Scan fails instead of returning SSIDs when radio is initialized
```

Equivalent accepted forms for the 3rd bracket (normalize before draft/create):

| User provides | Normalize to |
|---|---|
| `[SW_SQA_TC-11261]` | `SW_SQA_TC-11261` |
| `[https://siliconlabs.atlassian.net/browse/SW_SQA_TC-11261]` | `SW_SQA_TC-11261` |
| bare URL beside Summary | extract `SW_SQA_TC-11261` into the 3rd bracket in the draft Summary |

#### Summary `[product-or-device]` → Product(s)* and Platform Under Test (auto-fill)

Parse the first bracket tag and set both fields from `field-defaults.yaml` → `product_device_map`:

| Summary tag `[product-or-device]` | Product(s)* | Platform Under Test |
|---|---|---|
| `[SiWx917-Soc]` (also `SiWx917-SoC`) | `WiSeConnect-Soc` | `917 SoC` |
| `[SiWx917-NCP]` | `WiSeConnect-NCP` | `917 NCP` |
| `[SiWx917-RCP]` | `WiSeConnect-RCP` | `917 RCP` |

Rules:

- Prefer these three tags in Summary so Product(s)* and Platform Under Test can be set without asking.
- Matching is case-insensitive on the tag text.
- If the tag is missing from `product_device_map`, or `products` / `platform_under_test` is empty (e.g. bare `[SiWx917]`), **ask once** and suggest choosing `SiWx917-Soc` / `SiWx917-NCP` / `SiWx917-RCP` in the Summary.
- Do not invent Product(s)* or Platform Under Test values outside this map.
- Use the exact dropdown option string (e.g. `917 SoC`, including spacing/casing). On create, pass Platform Under Test via `additional_fields` (search with `jira_search_fields` for `Platform Under Test` if the custom field id is unknown).

#### Summary `[Affects Version]` → Affects Version/s and Reported Version (auto-fill)

Parse the **second** bracket tag and set both Jira fields to that exact version string:

| Summary tag | Jira field | Value |
|---|---|---|
| `[Affects Version]` (2nd bracket) | **Affects Version/s** | Same string as in Summary (e.g. `26Q2-GA`) |
| `[Affects Version]` (2nd bracket) | **Reported Version** | Same string as in Summary (e.g. `26Q2-GA`) |

Rules:

- The version text inside the second `[]` is user-defined (release name, SDK build, GA tag, etc.). Use it **verbatim** — do not invent or remap unless the user overrides.
- Both **Affects Version/s** and **Reported Version** get the **same** value from that tag.
- On create, set Affects Version/s via standard `versions` / `additional_fields` as required by the instance; set Reported Version via `additional_fields` (search with `jira_search_fields` for `Reported Version` if the custom field id is unknown). Match an existing Jira version name when the project requires a version object; if the version does not exist in the project, ask once rather than inventing.
- User override in the `SQA_JIRA` message for either field wins over the Summary tag.
<!-- DEFERRED (Dev team owns Fix Version/s after filing — do not ask or set):
- This Summary tag does **not** auto-fill **Fix Version/s** (still separate — ask or take from defaults when required).
-->
- Do **not** ask for or set **Fix Version/s** on create/update; SQA DevOps sets it after triage/assignment.

#### Summary `[Atlassian-TC-key]` → Repro Steps / Expected Results / labels (auto-fill)

Parse the **third** bracket as an **Atlassian Cloud test-case issue key** (typical project prefix `SW_SQA_TC`, e.g. `SW_SQA_TC-11261`).

| Summary tag | Fills |
|---|---|
| `[Atlassian-TC-key]` (3rd bracket) | Description → **Repro Steps** (from TC) |
| same | Description → **Expected** (prefer TC Expected Result; fall back to Summary `instead of` text) |
| same | Description → Test Case link (`https://siliconlabs.atlassian.net/browse/<KEY>`) |
| same | Labels / search hints from TC labels, components, and summary (optional `feature_label_map` enrichment) |
| same | After `FILE_JIRA` create: remote link / relates link to the TC when the Silabs Jira API allows |

**Atlassian Cloud only for TC lookup — do NOT use Silabs Jira MCP**

| Use for TC read | Do not use for TC read |
|---|---|
| **Atlassian Cloud** — `https://siliconlabs.atlassian.net` | Silabs Jira MCP (`user-jira-mcp` / `jira.silabs.com`) |
| Browse/API under `https://siliconlabs.atlassian.net/browse/<KEY>` | Assuming Silabs Server/DC holds `SW_SQA_TC-*` |

##### How to call Atlassian Cloud

1. Prefer the **Atlassian MCP** (`mcp.atlassian.com`) when it is enabled and authenticated in the user's Cursor MCP config.
2. If Atlassian MCP is unavailable, use the **Atlassian Cloud REST API** (`https://siliconlabs.atlassian.net/rest/api/3/...`) with the user's Atlassian Basic auth from `~/.cursor/mcp.json` (commented or active `atlassian` server `Authorization: Basic ...` header). Never print or echo the token in chat.
3. Fetch issue metadata with `GET /rest/api/3/issue/<KEY>?fields=summary,description,labels,components,...` (expand as needed).
4. **Repro Steps / Expected Result are NOT in the Jira description for Xray Tests.** They live in the Xray **Test details** panel:
   - UI path: issue → **Test details** → **Test Type** = `Steps` → table columns **Action** / **Data** / **Expected Result**
   - Repository breadcrumb (example): `Test Repository / 917_IoT_FreeRTOS / Wi-Fi STA / ...`
   - Prefer reading steps via Xray Cloud API / GraphQL when credentials exist; otherwise accept a screenshot/paste of the Steps table, or ask once for the full step list.
   - Do **not** treat empty `description` as “no steps” when Test Type is Steps — keep looking at the Steps table.
5. Do **not** invent steps or expected results when the Steps table cannot be read — ask once for a Steps screenshot/paste, and note the gap in the draft.

##### What to extract from the TC

Map TC content into the SI91X bug draft (do not overwrite user Summary observed/condition text):

| TC field / content | Bug draft field | Rules |
|---|---|---|
| Issue key + browse URL | Test Case / Linked TC | Always include in Description + Auto-fill provenance |
| Summary / title | Context; optional Labels enrichment | Do not replace bug Summary |
| Xray **Test details** → Test Type **Steps** → **Action** (+ **Data** if not `None`) | **Repro Steps** | Ordered step list; keep TC wording; prefix with “From `<KEY>` (Xray Steps):” |
| Same Steps table → **Expected Result** (per step + overall) | **Problem → Expected** | Prefer the TC’s overall/objective expected (or failing-step Expected Result); if empty, use Summary text after `instead of` |
| Labels / Components | Labels (optional) | Merge with `product_device_map` labels; if a label/component slug matches `feature_label_map`, add those mapped labels |
| Actual / failure notes on TC | Do **not** copy as bug Actual | Bug Actual always comes from Summary `<observed-behavior>` |

**Actual vs Expected precedence**

- **Actual** — always from Summary `<observed-behavior>` (user’s failure signature).
- **Expected** — TC Expected Result when present; else Summary text after `instead of`.
- If TC Expected and Summary Expected conflict, keep **both** in the draft (TC Expected as primary; note Summary Expected in provenance) and ask once only if they contradict in a way that would confuse triage.

#### Summary parse → auto-fill map

| Summary part | Example | Auto-fills |
|---|---|---|
| `[product-or-device]` (1st bracket) | `[SiWx917-Soc]` | **Product(s)\***; **Platform Under Test** (e.g. `917 SoC`); Labels via `product_device_map`; Environment → Board/Module hint |
| `[Affects Version]` (2nd bracket) | `[26Q2-GA]` | **Affects Version/s** and **Reported Version** (same value) |
| `[Atlassian-TC-key]` (3rd bracket) | `[SW_SQA_TC-11261]` | Repro Steps; Expected (from TC); TC browse link; Labels enrichment |
| `<observed-behavior>` | `DUT returns empty scan list` | Description → Actual |
| `<expected-behavior>` | `SSIDs` (after `instead of`) | Description → Expected (fallback if TC Expected empty) |
| `<condition>` | `associated and BG scan is enabled` (after `when`) | Description → Impact/condition; Repro Steps seed / failure context |

**Validation:** If Summary is missing any of the three bracket tags (`[product-or-device]`, `[Affects Version]`, `[Atlassian-TC-key]`), or does not contain `instead of` and `when`, reject and ask the user to resubmit using the preferred format. Do not invent a Summary.

**3rd-bracket validation:** The third tag must resolve to an Atlassian issue key matching `^[A-Z][A-Z0-9_]+-\d+$` (e.g. `SW_SQA_TC-11261`). Free-form feature names such as `[bg_scan]` are **rejected** — ask the user to resubmit with the Atlassian TC key or browse URL (example: `https://siliconlabs.atlassian.net/browse/SW_SQA_TC-11261`).

### Mandatory user inputs (ask if missing)

Only these require user action. Collect them before creating the issue.

| # | Field / input | How user provides | Notes |
|---|---|---|---|
| 1 | **Summary\*** | Preferred format above | Mandatory; drives Product(s)*, Platform Under Test, Affects/Reported Version, and TC lookup |
| 2 | **Product(s)\*** | Prefer via Summary tag | Auto from `[SiWx917-Soc\|NCP\|RCP]`; ask only if tag unmapped |
| 3 | **Platform Under Test** | Prefer via Summary tag | Auto e.g. `[SiWx917-Soc]` → `917 SoC`; ask only if tag unmapped |
| 4 | **Affects Version/s** | Prefer via Summary 2nd tag | Same value as Summary `[Affects Version]` |
| 5 | **Reported Version** | Prefer via Summary 2nd tag | Same value as Summary `[Affects Version]` |
| 6 | **Component/s\*** | Always auto | Always `Bug_Triage_Team` — never ask |
| 7 | **Reporter\*** | Auto from OS username → Jira profile | Resolve via `jira_get_user_profile` using OS `USERNAME` (or local `reporter_identifier` override) — never invent; ask once if lookup fails |
| 8 | **IsCrash\*** | Always auto | Default `FALSE`; may become `TRUE` after Summary + log analysis (see IsCrash*) |
| 9 | **Issue Reproducibility\*** | Always auto | Always `Easily reproducible` unless user overrides |
| 10 | **Issue Type** | Select if not Bug | Default: `Bug` from `field-defaults.yaml` |
| 11 | **Project** | Select if not default | Default: `SI91X` from `field-defaults.yaml` |
<!-- DEFERRED (Dev team owns Fix Version/s — do not ask or set):
| 12 | **Fix Version/s** | Select or paste | Still separate from Affects/Reported; ask if blank in defaults |
-->
| 12 | **Atlassian TC key** | Summary 3rd bracket (or URL) | Mandatory; used to pull Repro Steps + Expected from Atlassian Cloud |
| 13 | **Evidence** | Path to text logs / pytest report / cmds file, or paste | Screenshots alone are not enough; required for IsCrash* upgrade to TRUE |
| 14 | **Repro frequency** (optional but recommended) | e.g. `10/10` | Defaults to `unknown` if omitted |

Do **not** ask for fields that can be derived from Summary + Atlassian TC + defaults unless creation would fail without them.

### Auto-filled / sourced fields (do not ask by default)

| Jira / body field | Source priority |
|---|---|
| Summary | User (mandatory format; 3rd bracket = Atlassian TC key) |
| **Product(s)\*** | Summary `[product-or-device]` → `product_device_map.products` (`WiSeConnect-Soc` / `WiSeConnect-NCP` / `WiSeConnect-RCP`) |
| **Platform Under Test** | Summary `[product-or-device]` → `product_device_map.platform_under_test` (e.g. `[SiWx917-Soc]` → `917 SoC`) |
| **Affects Version/s** | Summary 2nd bracket `[Affects Version]` — verbatim |
| **Reported Version** | Same as Summary `[Affects Version]` — verbatim |
| **Component/s\*** | Always `Bug_Triage_Team` from `field-defaults.yaml` → `component` (never from Summary; never ask) |
| **Reporter\*** | Auto: OS username → `jira_get_user_profile` (see below); optional local `reporter_identifier` — mandatory on every draft/create |
| **IsCrash\*** | Default `FALSE` from `field-defaults.yaml` → `is_crash`; set `TRUE` only per IsCrash* rules below |
| **Issue Reproducibility\*** | Always `Easily reproducible` from `field-defaults.yaml` → `issue_reproducibility` (never ask; user override wins) |
| Issue Type | `field-defaults.yaml` → Bug |
| Project | `field-defaults.yaml` → SI91X |
| Labels | Summary product map + Atlassian TC labels/components (+ optional `feature_label_map` enrichment) |
| Description → Actual / Condition | Parsed from Summary |
| Description → Expected | Atlassian TC Expected Result; fallback = Summary after `instead of` |
| Description → Repro Steps | Atlassian TC steps (required when TC is readable); seed with Summary condition if TC steps empty |
| Environment (board hint) | Summary product tag; refine from TC / user evidence |
| FW / SDK / Release | User evidence / ask once (do not invent from TC unless TC explicitly states the failing build) |
| Toolchain | Evidence logs or ask once |
| Build provenance | Repo git context or packaged artifact from evidence; ask once if unclear |
| Baseline (last known good / failing) | User evidence paths; ask once if missing |
| Assignee | `field-defaults.yaml` or leave unassigned |
| Linked test case | Summary `[Atlassian-TC-key]` → Cloud browse URL; remote link after create when possible |

### Agent workflow for `SQA_JIRA`

1. Confirm the query contains `SQA_JIRA` (or the skill was attached).
2. Load `field-defaults.yaml` from this skill folder.
3. Require Summary in the preferred format; parse product, **affects version**, **Atlassian TC key**, observed, expected, condition. Normalize URL → issue key for the 3rd bracket.
4. Resolve **Product(s)\*** and **Platform Under Test** from `[product-or-device]` via `product_device_map`; ask once only if either is unmapped/empty.
5. Set **Affects Version/s** and **Reported Version** from Summary `[Affects Version]` (same value); ask once only if the version cannot be applied in Jira.
6. Set **Component/s\*** to `Bug_Triage_Team` always (from `field-defaults.yaml` → `component`).
7. Resolve **Reporter\*** automatically from OS username → `jira_get_user_profile` (mandatory — see Reporter* section below).
8. Resolve **IsCrash\*** (mandatory — default `FALSE`; see IsCrash* section below). Analyze attached logs when Summary mentions crash.
9. Set **Issue Reproducibility\*** to `Easily reproducible` always (from `field-defaults.yaml` → `issue_reproducibility`), unless the user overrides.
10. **Fetch the Atlassian Cloud TC** for the 3rd-bracket key (see Summary `[Atlassian-TC-key]`). Merge Repro Steps + Expected into the draft. Do **not** ask for or set **Fix Version/s** (SQA DevOps owns it after filing).
11. Collect only missing **mandatory** items (evidence; TC key if invalid/unreadable).
12. Draft the full ticket body using the template below (Actual from Summary; Expected from TC when available; Repro Steps from TC). Include Product(s)*, Platform Under Test, Affects Version/s, Reported Version, Component/s*, Reporter*, IsCrash*, Issue Reproducibility*, and Atlassian TC key/URL in the draft provenance.
13. Run the **Mandatory Jira ticket checklist** (quality gates). List any remaining gaps explicitly.
14. **STOP and wait for `FILE_JIRA`** (hard gate — see **Draft approval before create** below). Do **not** call `jira_create_issue` / `jira_update_issue` in this same turn after first presenting the draft.
15. Only after the engineer replies with **`FILE_JIRA`**: create or update the SI91X issue via Silabs Jira MCP. Description body must start with the Cursor disclaimer (see **Cursor disclaimer**). Set Product(s)* and Platform Under Test from the map, Affects Version/s + Reported Version from Summary, Component/s* to `Bug_Triage_Team`, Reporter* from the resolved profile, IsCrash* (`FALSE` or `TRUE`), and Issue Reproducibility* to `Easily reproducible` (or user override). Add a remote link to `https://siliconlabs.atlassian.net/browse/<TC-KEY>` when possible (`jira_create_remote_issue_link`).
16. Return: issue key/URL, what was auto-filled (including Product(s)*, Platform Under Test, Affects Version/s, Reported Version, Component/s*, Reporter*, IsCrash*, Issue Reproducibility*, Atlassian TC), and what the user still must attach if evidence was path-only.

### Draft approval before create (mandatory hard gate)

**Never create or update a Jira issue until the engineer provides explicit approval through the `FILE_JIRA` command** after a draft was shown in this conversation.

Required sequence:

1. Build and **show the full draft** (field table + Description template + gaps + auto-fill provenance).
2. Ask explicitly for filing approval using this exact prompt:

   > Reply **`FILE_JIRA`** to create this ticket in Jira, or send edits to revise the draft.

3. **End the turn.** Do not call `jira_create_issue`, `jira_update_issue`, or any other Jira write MCP tool yet.
4. In a **later** user message, create **only** if that message contains the exact command **`FILE_JIRA`** (case-sensitive) and refers to the draft already shown (or the latest revised draft).

**Only approval that counts (after the draft was shown):**

- `FILE_JIRA` (exact token, case-sensitive)

Optional: `FILE_JIRA` may appear with light surrounding text (e.g. `FILE_JIRA please` / `ok FILE_JIRA`) as long as the token is present. Prefer the bare command.

**Does NOT count as approval** (draft + wait still required — do **not** file):

- `create a JIRA` / `create JIRA` / `file a bug` / `SQA_JIRA create now` / `approve` / `approved`
- Attaching `/sqa-jira-quality-guideline` alone
- Providing only a Summary line
- `SQA_JIRA` with Summary but without a later `FILE_JIRA`
- Implicit intent (“please create”, “looks good, create it”, “file it”) in the same message that first supplies the Summary, or in any message before a draft was shown
- Soft confirmations that omit the `FILE_JIRA` token

If the engineer edits the draft, show an updated draft and require a new **`FILE_JIRA`** before create.

If the engineer asks to create without having seen a draft in this chat, **show the draft first** and wait for `FILE_JIRA` — do not file immediately.


### Reporter* (mandatory — auto-detect OS user → Jira profile)

There is no Jira MCP “get current user” tool. Auto-detect the filer and resolve via `jira_get_user_profile` — do **not** hardcode a person in the shared skill.

1. Identifier source (first match wins):
   - User override in the `SQA_JIRA` message (username / email / key)
   - Non-empty `field-defaults.yaml` → `reporter_identifier` (optional local override only — leave blank in the shared/repo copy)
   - Else **auto-detect OS username** (Windows `USERNAME` / `$env:USERNAME`, or Unix `USER` / `whoami`) as the lookup hint
2. Call `jira_get_user_profile` with that identifier (typical Silabs usernames match the Windows login, e.g. `viverma`).
3. From the profile response:
   - Show **display_name** in the draft (e.g. the engineer’s real name)
   - Set Jira **Reporter\*** using profile **`name`** (Server/DC username) or **`key`** (`JIRAUSER…`) as required by the instance — prefer `name` when present
4. On create, pass Reporter via `additional_fields`, e.g. `{"reporter": {"name": "<profile.name>"}}` (adjust shape if the MCP/instance requires `key` instead).
5. If lookup fails (OS username ≠ Jira username), **ask once** for Jira username or email; optionally suggest setting a local `reporter_identifier` in `field-defaults.yaml`. Do not invent a reporter; do not create the issue until Reporter* is resolved.

Do not leave Reporter* blank when drafting. Do not use Assignee as a substitute for Reporter*. Do not fall back to another engineer’s username from chat history or prior tickets.

### IsCrash* (mandatory — default FALSE; log-backed TRUE)

Always set **IsCrash\*** on every draft/create. Allowed values: `FALSE` | `TRUE` (use these exact strings unless Jira field options differ — then map to the closest allowed option).

**Default:** `FALSE` (`field-defaults.yaml` → `is_crash`).

**Upgrade to `TRUE` only when both are true:**

1. **Summary mentions crash** (case-insensitive) in observed behavior, expected, or condition — e.g. `crash`, `crashed`, `crashing`, `HardFault`, `hard fault`, `fault`, `assert`, `assertion`, `panic`, `watchdog reset`, `WDT`, `core dump`, `coredump`, `exception` (CPU/fault sense, not HTTP).
2. **Attached log analysis finds crash/dump evidence** in textual runtime/UART/pytest logs or dump files the user provided (paths or pasted text). Treat as positive evidence if any of the following appear (non-exhaustive):
   - Stack backtrace / call stack / `PC:` / `LR:` / `HardFault` / `MemManage` / `BusFault` / `UsageFault`
   - `assert` / `ASSERT` / `abort` / `Fatal error` / `Guru Meditation` / panic banner
   - Core dump / coredump / `.dmp` / crash dump / register dump / fault status registers (`CFSR`, `HFSR`, `BFAR`, `MMFAR`)
   - Watchdog reset tied to a fault window, or MCU reset reason indicating crash/WDT after a fault log

**Rules:**

- Summary says crash but **no logs attached** → keep `FALSE`, ask once for textual logs/dump, and note IsCrash* provisional until evidence is analyzed.
- Summary says crash, logs attached, but **no crash/dump signatures found** → keep `FALSE`; state in draft provenance that Summary claimed crash but logs lack dump/fault evidence.
- Summary does **not** mention crash → keep `FALSE` even if logs look noisy; do not upgrade from logs alone unless the user explicitly overrides IsCrash*=TRUE.
- User explicit override in the `SQA_JIRA` message (`IsCrash=TRUE` / `IsCrash=FALSE`) wins over auto logic.
- On create, pass IsCrash* via `additional_fields` using the project’s field id/name (search with `jira_search_fields` for `IsCrash` if unknown). Do not invent the custom field id.

Cite the matching log lines/filenames briefly in Auto-fill provenance when setting `TRUE`.

### Issue Reproducibility* (mandatory — default Easily reproducible)

Always set **Issue Reproducibility\*** on every draft/create.

**Default:** `Easily reproducible` (`field-defaults.yaml` → `issue_reproducibility`).

Rules:

- Do not ask the user for this field by default.
- User override in the `SQA_JIRA` message wins (e.g. intermittent / Difficult to reproduce — use the exact Jira option name they provide).
- On create, pass via `additional_fields` using the project’s field id/name (search with `jira_search_fields` for `Issue Reproducibility` if unknown). Do not invent the custom field id.

### Atlassian Cloud TC lookup (mandatory from Summary 3rd bracket)

Always attempt this when drafting a `SQA_JIRA` ticket.

1. Resolve issue key from the 3rd bracket or pasted browse URL.
2. Read the issue from **Atlassian Cloud** only (MCP or REST — see above). Never use Silabs `user-jira-mcp` for `SW_SQA_TC-*`.
3. Populate:
   - **Repro Steps** from TC steps / description
   - **Expected** from TC Expected Result (fallback: Summary `instead of` text)
   - TC link in Description + provenance
   - Optional labels from TC labels/components (+ `feature_label_map` enrichment)
4. If lookup fails (auth, missing key, empty steps): continue draft with Summary-only Expected/condition seed, list the gap, and ask once for a readable TC URL or pasted steps. Do **not** invent TC content.

Legacy Silabs Xray/TestRail keys (`C990068`, etc.) are **not** a substitute for the Summary 3rd bracket. If the user also pastes a legacy key, treat it as optional extra context only; the Atlassian TC key remains mandatory in Summary.

---

## Top recurring quality gaps (with examples)

| Common gap | Example Jira IDs |
|---|---|
| Screenshot-only evidence, no textual logs | SI91X-21909, SI91X-21927, SI91X-21930, SI91X-21933, SI91X-21936, SI91X-21924, SI91X-21945, SI91X-21947 |
| Missing exact repro command sequence | SI91X-21909, SI91X-20953, SI91X-21933, SI91X-21936, SI91X-21924 |
| Missing failing vs passing baseline logs | SI91X-20953, SI91X-21902, SI91X-21907, SI91X-21805, SI91X-21900 |
| Missing bug-type-specific artifacts (map/pcap/MTBF, etc.) | SI91X-21900, SI91X-21907, SI91X-21805, SI91X-21912, SI91X-21798, SI91X-21947, SI91X-21936 |
| Attachment format/decoding friction | SI91X-21907, SI91X-21805, SI91X-21912, SI91X-21900, SI91X-21798, SI91X-21944 |
| Symptom/evidence mismatch in same ticket | SI91X-21933 |
| Missing source/provenance context (package vs repo, exact source path) | SI91X-21924, SI91X-21933 |
<!-- DEFERRED Fix Version/s from this gap row; keep Affects/Reported:
| Release targeting ambiguity (`Fix Version/s` / `Affects Version/s` / `Reported Version`) | SI91X-22058, SI91X-22065 |
-->
| Release targeting ambiguity (`Affects Version/s` / `Reported Version`) | SI91X-22058, SI91X-22065 |

> Note: Tooling/auth issues (for example GitHub MCP access) are process/tooling constraints, not reporter-quality issues.

---

## Mandatory Jira ticket checklist (SQA)

### 1) Problem definition
- One clear failure signature per ticket
- Expected behavior vs actual behavior (Actual from Summary; Expected from Atlassian TC when available)
- Impact and severity context

### 2) Reproduction
- Exact repro steps (ordered) — prefer Atlassian TC steps
- Exact commands/config used (copy-paste text)
- Repro frequency (for example 10/10, 2/10)
- Linked Atlassian TC key/URL in Description

### 3) Environment and build provenance
- Board/module/device details
- FW version, SDK/release, toolchain (GCC/LLVM/LTO)
- Source type: packaged SDK vs repo checkout
- If packaged: artifact ID/tag/hash
- If repo: repo + branch + commit SHA

### 4) Logs and evidence (text required)
- Full textual runtime/UART logs
- Full textual build/compiler logs (for build failures)
- Do not rely on screenshots alone

### 5) Baseline comparison
- Last known good version/build
- Failing version/build
- Side-by-side observation summary

### 6) Bug-type-specific artifacts
- Linker/memory issues: map file + symbol evidence
- Cloud/network issues: pcap/sniffer + request/response timeline
- MTBF/long-run issues: fail-window logs with phase markers/timestamps
- Flashing/programming issues: commander/J-Link full textual traces

### 7) Correct issue routing metadata
- Correct issue type (Bug vs Improvement/Doc/Support)
- Populate **Product(s)\*** from Summary `[product-or-device]` (`WiSeConnect-Soc` / `WiSeConnect-NCP` / `WiSeConnect-RCP`)
- Populate **Platform Under Test** from Summary `[product-or-device]` (e.g. `[SiWx917-Soc]` → `917 SoC`)
- Populate **Affects Version/s** and **Reported Version** from Summary `[Affects Version]` (same value)
- Populate **Component/s\*** always as `Bug_Triage_Team`
- Populate **Reporter\*** automatically from OS username → `jira_get_user_profile` (not a hardcoded teammate)
- Populate **IsCrash\*** (`FALSE` by default; `TRUE` only if Summary mentions crash and attached logs show crash/dump evidence)
- Populate **Issue Reproducibility\*** always as `Easily reproducible` (unless user overrides)
- Include Atlassian TC key/URL from Summary 3rd bracket; pull Repro Steps / Expected from Cloud
<!-- DEFERRED (SQA DevOps owns Fix Version/s after filing — do not ask or set):
- Populate `Fix Version/s` when required (separate from Affects/Reported)
-->

---

## Strongly recommended ticket template (paste into Jira)

````markdown
> **Disclaimer:** This triage and proposed PR is Cursor generated. Recommend manual review before accepting this fix.

## Summary
`[<product-or-device>][Affects Version][<Atlassian-TC-key>] <observed-behavior> instead of <expected-behavior> when <condition>`

## Test Case
- Key: <SW_SQA_TC-#####>
- URL: https://siliconlabs.atlassian.net/browse/<SW_SQA_TC-#####>

## Problem
- Expected: <from Atlassian TC Expected Result; else Summary after "instead of">
- Actual: <from Summary — observed-behavior>
- Condition: <from Summary — after "when">
- Impact:

## Repro Steps
<!-- Prefer ordered steps copied from Atlassian TC <KEY> -->
1)
2)
3)

## Exact Commands/Config
```text
<paste exact commands>
```

## Environment
- Board/Module: <from [product-or-device] + evidence>
- FW:
- SDK/Release:
- Toolchain:
- Host OS/Tools:

## Build Provenance
- Source type: packaged SDK | repo checkout
- Artifact/tag/hash OR branch+commit:

## Baseline
- Last known good:
- Failing build:
- Difference observed:

## Evidence Attached
- [ ] Text runtime logs
- [ ] Text build logs
- [ ] Map/symbol files (if linker/memory)
- [ ] Pcap/sniffer (if cloud/network)
- [ ] MTBF fail-window logs (if long-run)

## Auto-fill provenance
- Summary parse: product= / affects_version= / atlassian_tc=
- Product(s)*: <WiSeConnect-Soc | WiSeConnect-NCP | WiSeConnect-RCP>
- Platform Under Test: <917 SoC | 917 NCP | 917 RCP>
- Affects Version/s: <from Summary [Affects Version]>
- Reported Version: <same as Affects Version>
- Component/s*: Bug_Triage_Team
- Reporter*: <display_name> (<name> / <key> from jira_get_user_profile)
- IsCrash*: FALSE | TRUE (reason: default | summary+log evidence | user override; cite log markers if TRUE)
- Issue Reproducibility*: Easily reproducible (or user override)
- Defaults file: field-defaults.yaml
- Atlassian TC: <KEY> (<browse URL>; Repro/Expected source: TC | Summary fallback | unavailable)
````

---

## Definition of triage-ready ticket

A ticket is triage-ready when it has:

- Summary in the preferred format (3rd bracket = Atlassian TC key),
- reproducible steps (from the Atlassian TC when available),
- full textual logs,
- clear environment/build provenance,
- passing vs failing baseline,
- and bug-specific artifacts needed to validate root cause.

If these are missing, triage will likely remain provisional and PR creation may be delayed.
