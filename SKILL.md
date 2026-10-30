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
3. **Xray / linked test case** (when an Xray/TestRail ID or test key is provided)
4. **Skill defaults** in `field-defaults.yaml` (same folder as this skill)
5. **Ask once** only if still missing and required to create the issue

**Hard gate:** Always show a draft and wait for explicit user approval before any `jira_create_issue` / create-side update. Phrases like “create a JIRA” or a Summary alone are **not** approval — see **Draft approval before create**.

### Phase 1 — start with Summary*

User must provide **Summary\*** in this preferred format (use verbatim):

`[<product-or-device>][Affects Version][<feature>] <observed-behavior> instead of <expected-behavior> when <condition>`

Examples:

```text
[SiWx917-Soc][26Q2-GA][bg_scan] DUT returns empty scan list instead of SSIDs when associated and BG scan is enabled
[SiWx917-NCP][3.5.0][connect] Association fails with 0x10003 instead of success when AP uses WPA3-SAE only
[SiWx917-RCP][26Q1][scan] Scan fails instead of returning SSIDs when radio is initialized
```

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
- This Summary tag does **not** auto-fill **Fix Version/s** (still separate — ask or take from defaults/Xray when required).
-->
- Do **not** ask for or set **Fix Version/s** on create/update; SQA DevOps sets it after triage/assignment.

#### Summary parse → auto-fill map

| Summary part | Example | Auto-fills |
|---|---|---|
| `[product-or-device]` (1st bracket) | `[SiWx917-Soc]` | **Product(s)\***; **Platform Under Test** (e.g. `917 SoC`); Labels via `product_device_map`; Environment → Board/Module hint |
| `[Affects Version]` (2nd bracket) | `[26Q2-GA]` | **Affects Version/s** and **Reported Version** (same value) |
| `[feature]` (3rd bracket) | `[bg_scan]` | Labels via `feature_label_map`; Description section headers; search hints for similar bugs / Xray cases |
| `<observed-behavior>` | `DUT returns empty scan list` | Description → Actual |
| `<expected-behavior>` | `SSIDs` (after `instead of`) | Description → Expected |
| `<condition>` | `associated and BG scan is enabled` (after `when`) | Description → Impact/condition; Repro Steps seed |

**Validation:** If Summary is missing any of the three bracket tags (`[product-or-device]`, `[Affects Version]`, `[feature]`), or does not contain `instead of` and `when`, reject and ask the user to resubmit using the preferred format. Do not invent a Summary.

### Mandatory user inputs (ask if missing)

Only these require user action. Collect them before creating the issue.

| # | Field / input | How user provides | Notes |
|---|---|---|---|
| 1 | **Summary\*** | Preferred format above | Mandatory; drives Product(s)*, Platform Under Test, Affects/Reported Version, and most auto-fill |
| 2 | **Product(s)\*** | Prefer via Summary tag | Auto from `[SiWx917-Soc\|NCP\|RCP]`; ask only if tag unmapped |
| 3 | **Platform Under Test** | Prefer via Summary tag | Auto e.g. `[SiWx917-Soc]` → `917 SoC`; ask only if tag unmapped |
| 4 | **Affects Version/s** | Prefer via Summary 2nd tag | Same value as Summary `[Affects Version]` |
| 5 | **Reported Version** | Prefer via Summary 2nd tag | Same value as Summary `[Affects Version]` |
| 6 | **Component/s\*** | Always auto | Always `Bug_Triage_Team` — never ask |
| 7 | **Reporter\*** | Always from Jira profile | Resolve via `jira_get_user_profile` — never invent; ask only if lookup fails |
| 8 | **IsCrash\*** | Always auto | Default `FALSE`; may become `TRUE` after Summary + log analysis (see IsCrash*) |
| 9 | **Issue Reproducibility\*** | Always auto | Always `Easily reproducible` unless user overrides |
| 10 | **Issue Type** | Select if not Bug | Default: `Bug` from `field-defaults.yaml` |
| 11 | **Project** | Select if not default | Default: `SI91X` from `field-defaults.yaml` |
<!-- DEFERRED (Dev team owns Fix Version/s — do not ask or set):
| 12 | **Fix Version/s** | Select or paste | Still separate from Affects/Reported; ask if blank in defaults and not found via Xray |
-->
| 12 | **Evidence** | Path to text logs / pytest report / cmds file, or paste | Screenshots alone are not enough; required for IsCrash* upgrade to TRUE |
| 13 | **Repro frequency** (optional but recommended) | e.g. `10/10` | Defaults to `unknown` if omitted |
| 14 | **Xray / TestRail / test key** (optional) | e.g. `C990068` or Xray issue key | When present, pull env, steps, release from Xray |

Do **not** ask for fields that can be derived from Summary + defaults + Xray unless creation would fail without them.

### Auto-filled / sourced fields (do not ask by default)

| Jira / body field | Source priority |
|---|---|
| Summary | User (mandatory format) |
| **Product(s)\*** | Summary `[product-or-device]` → `product_device_map.products` (`WiSeConnect-Soc` / `WiSeConnect-NCP` / `WiSeConnect-RCP`) |
| **Platform Under Test** | Summary `[product-or-device]` → `product_device_map.platform_under_test` (e.g. `[SiWx917-Soc]` → `917 SoC`) |
| **Affects Version/s** | Summary 2nd bracket `[Affects Version]` — verbatim |
| **Reported Version** | Same as Summary `[Affects Version]` — verbatim |
| **Component/s\*** | Always `Bug_Triage_Team` from `field-defaults.yaml` → `component` (never from Summary; never ask) |
| **Reporter\*** | Jira profile via `jira_get_user_profile` (see below) — mandatory on every draft/create |
| **IsCrash\*** | Default `FALSE` from `field-defaults.yaml` → `is_crash`; set `TRUE` only per IsCrash* rules below |
| **Issue Reproducibility\*** | Always `Easily reproducible` from `field-defaults.yaml` → `issue_reproducibility` (never ask; user override wins) |
| Issue Type | `field-defaults.yaml` → Bug |
| Project | `field-defaults.yaml` → SI91X |
| Labels | Summary product + feature maps → defaults |
| Description → Expected / Actual / Condition | Parsed from Summary |
| Description → Repro Steps (seed) | Condition + Xray steps if linked |
| Environment (board hint) | Summary product tag; refine from Xray / user evidence |
| FW / SDK / Release | Xray test execution / user evidence / ask once |
| Toolchain | Evidence logs or ask once |
| Build provenance | Repo git context or packaged artifact from evidence; ask once if unclear |
| Baseline (last known good / failing) | User evidence paths or Xray pass/fail history; ask once if missing |
| Assignee | `field-defaults.yaml` or leave unassigned |
| Epic / parent | Only if user or Xray linkage provides it |

### Agent workflow for `SQA_JIRA`

1. Confirm the query contains `SQA_JIRA`.
2. Load `field-defaults.yaml` from this skill folder.
3. Require Summary in the preferred format; parse product, **affects version**, feature, observed, expected, condition.
4. Resolve **Product(s)\*** and **Platform Under Test** from `[product-or-device]` via `product_device_map`; ask once only if either is unmapped/empty.
5. Set **Affects Version/s** and **Reported Version** from Summary `[Affects Version]` (same value); ask once only if the version cannot be applied in Jira.
6. Set **Component/s\*** to `Bug_Triage_Team` always (from `field-defaults.yaml` → `component`).
7. Resolve **Reporter\*** from Jira user profile (mandatory — see Reporter* section below).
8. Resolve **IsCrash\*** (mandatory — default `FALSE`; see IsCrash* section below). Analyze attached logs when Summary mentions crash.
9. Set **Issue Reproducibility\*** to `Easily reproducible` always (from `field-defaults.yaml` → `issue_reproducibility`), unless the user overrides.
10. Collect only missing **mandatory** items from the table above (evidence). Do **not** ask for or set **Fix Version/s** (SQA DevOps owns it after filing).
<!-- DEFERRED: (Fix Version/s if required, evidence). -->
11. If an Xray/TestRail/test key is present, query Jira/Xray and merge: release version, repository path, documented steps, last execution result (do not overwrite Summary Affects/Reported Version unless user asks).
12. Draft the full ticket body using the template below (Expected/Actual from Summary; do not leave them blank). Include Product(s)*, Platform Under Test, Affects Version/s, Reported Version, Component/s*, Reporter*, IsCrash*, and Issue Reproducibility* in the draft provenance.
13. Run the **Mandatory Jira ticket checklist** (quality gates). List any remaining gaps explicitly.
14. **STOP and wait for `FILE_JIRA`** (hard gate — see **Draft approval before create** below). Do **not** call `jira_create_issue` / `jira_update_issue` in this same turn after first presenting the draft.
15. Only after the engineer replies with **`FILE_JIRA`**: create or update the Jira issue via Jira MCP. Set Product(s)* and Platform Under Test from the map, Affects Version/s + Reported Version from Summary, Component/s* to `Bug_Triage_Team`, Reporter* from the resolved profile, IsCrash* (`FALSE` or `TRUE`), and Issue Reproducibility* to `Easily reproducible` (or user override).
16. Return: issue key/URL, what was auto-filled (including Product(s)*, Platform Under Test, Affects Version/s, Reported Version, Component/s*, Reporter*, IsCrash*, Issue Reproducibility*), and what the user still must attach if evidence was path-only.

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


### Reporter* (mandatory — from Jira profile)

There is no Jira MCP “get current user” tool. Always resolve the filer via `jira_get_user_profile`:

1. Identifier source (first match wins):
   - User override in the `SQA_JIRA` message (username / email / key)
   - `field-defaults.yaml` → `reporter_identifier`
   - Else OS username (e.g. Windows `USERNAME`) as a lookup hint
2. Call `jira_get_user_profile` with that identifier.
3. From the profile response:
   - Show **display_name** in the draft (e.g. `Vivek Verma`)
   - Set Jira **Reporter\*** using profile **`name`** (Server/DC username, e.g. `viverma`) or **`key`** (`JIRAUSER…`) as required by the instance — prefer `name` when present
4. On create, pass Reporter via `additional_fields`, e.g. `{"reporter": {"name": "<profile.name>"}}` (adjust shape if the MCP/instance requires `key` instead).
5. If lookup fails, **ask once** for Jira username or email; do not invent a reporter; do not create the issue until Reporter* is resolved.

Do not leave Reporter* blank when drafting. Do not use Assignee as a substitute for Reporter*.

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

### Xray lookup (when ID provided)

Use Jira/Xray-backed fields when available (names may vary by project; search fields if needed):

<!-- DEFERRED (do not use Xray to drive Fix Version/s):
- Release / Fix version candidates
-->
- Release / Affects Version candidates (for Summary / Affects / Reported only — not Fix Version/s)
- Test repository path / Technology / Product line
- Documented test steps → Repro Steps
- Last execution status → Baseline seed (pass vs fail)

If Xray is unavailable, continue with Summary + defaults + user evidence; note the gap in the reply. Do not invent Xray data.

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
- Expected behavior vs actual behavior (must match Summary parse)
- Impact and severity context

### 2) Reproduction
- Exact repro steps (ordered)
- Exact commands/config used (copy-paste text)
- Repro frequency (for example 10/10, 2/10)

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
- Populate **Reporter\*** from Jira user profile (`jira_get_user_profile`)
- Populate **IsCrash\*** (`FALSE` by default; `TRUE` only if Summary mentions crash and attached logs show crash/dump evidence)
- Populate **Issue Reproducibility\*** always as `Easily reproducible` (unless user overrides)
<!-- DEFERRED (SQA DevOps owns Fix Version/s after filing — do not ask or set):
- Populate `Fix Version/s` when required (separate from Affects/Reported)
-->

---

## Strongly recommended ticket template (paste into Jira)

````markdown
## Summary
`[<product-or-device>][Affects Version][<feature>] <observed-behavior> instead of <expected-behavior> when <condition>`

## Problem
- Expected: <from Summary — after "instead of">
- Actual: <from Summary — observed-behavior>
- Condition: <from Summary — after "when">
- Impact:

## Repro Steps
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
- Summary parse: product= / affects_version= / feature=
- Product(s)*: <WiSeConnect-Soc | WiSeConnect-NCP | WiSeConnect-RCP>
- Platform Under Test: <917 SoC | 917 NCP | 917 RCP>
- Affects Version/s: <from Summary [Affects Version]>
- Reported Version: <same as Affects Version>
- Component/s*: Bug_Triage_Team
- Reporter*: <display_name> (<name> / <key> from jira_get_user_profile)
- IsCrash*: FALSE | TRUE (reason: default | summary+log evidence | user override; cite log markers if TRUE)
- Issue Reproducibility*: Easily reproducible (or user override)
- Defaults file: field-defaults.yaml
- Xray/TestRail: <id or none>
````

---

## Definition of triage-ready ticket

A ticket is triage-ready when it has:

- Summary in the preferred format,
- reproducible steps,
- full textual logs,
- clear environment/build provenance,
- passing vs failing baseline,
- and bug-specific artifacts needed to validate root cause.

If these are missing, triage will likely remain provisional and PR creation may be delayed.
