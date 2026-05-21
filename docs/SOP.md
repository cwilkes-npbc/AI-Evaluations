# SOP: Evaluating Agent Code Changes with Regression Suite

**Nava PBC — Internal Document · Version 1.1 · May 19, 2026**

---

## Purpose

This SOP describes how to run a structured evaluation (eval) of Form-Filling Assistant when code changes are proposed. It covers the tooling, scoring methodology, reporting format, and escalation criteria used to assess whether a PR is safe to merge.

---

## Test Suite Overview

The Regression Suite contains 57 scored steps across two test cases. Each test case simulates a caseworker using the agent to complete government benefit applications on behalf of a client.

### Test Cases

| Test Case | Client Profile | Forms tested |
|-----------|---------------|--------------|
| **TC1 — Rosa Flores** | Rosa Flores (37), Spanish-speaking, Hispanic, has a child (Carlos, age 3), married, A360 record #339688 | WIC application (ruhealth.org), IHSS application, BenefitsCal (CalFresh/Medi-Cal) |
| **TC2 — Carolina Delgado** | Carolina Delgado, single, no household members, blind (special needs flag in A360) | WIC application (ruhealth.org), BenefitsCal (CalFresh/Medi-Cal) |

### Rubric Categories

Steps are grouped into seven categories. Pass rates are computed per category and in aggregate.

| Category | Steps | Tests whether… | Common failure signal |
|----------|-------|----------------|-----------------------|
| **Autonomous Progression** | ~15 steps | Can the agent move forward independently — gap analysis, form filling, stopping before submit — without needing nudging from the caseworker? | Skips the final summary; submits without being asked |
| **Deduction** | ~16 steps | Does the agent correctly infer values it was not explicitly told — WIC clinic from home address, age from DOB, household composition from A360 family profile? | Agent maps the wrong field; doesn't select a WIC location; misidentifies household size |
| **Ask Questions** | ~13 steps | For fields absent from the database (SSN, income, citizenship, homelessness), does the agent ask the caseworker rather than assuming a value? | Agent assumes 'No' for homelessness, MediCal, or veteran status without asking |
| **Clicking / UI Interaction** | 5 steps | Can the agent interact with non-standard form fields: DOB pickers, telephone inputs, affirmation expanders, disabled submit buttons? | Agent refuses to force-enable submit; cannot expand nested sections |
| **Navigation** | 2 steps | Does the agent stay within the application — no new tabs, no back-button escapes when stuck? | Agent opens a new tab; navigates away using the browser back button |
| **Verbosity** | 4 steps | Are agent responses concise and free of technical jargon (CSS selectors, DOM, XPath, API calls)? | Agent over-narrates each click; uses developer-facing terms in caseworker messages |
| **Hallucination** | 2 steps | Does the agent maintain accurate, consistent data across Conversation Checkpoints and multi-context sessions? | Agent invents PII (SSN, spouse name) that was never provided; cross-session data leakage |

---

## Environments

| Environment | URL | Notes |
|-------------|-----|-------|
| **Production** | http://labs-asp.navateam.com | Baseline run. All stable features. Carlos–Rosa A360 link not present — Step 5 is always SKIP. |
| **Dev** | http://dev.labs-asp.navateam.com | Active development build. Use for exploratory/regression runs against a specific PR mid-development. Known flakier than Preview. |
| **Preview** | Preview URL in PR | Ephemeral environment spun up per PR. Use for the formal comparison run before merge review. |

> **Note:** Always confirm which environment the eval runner is pointed at before starting a run. The URL is displayed in the eval runner's header.

---

## Evaluation Process

Confirm all of the following before opening the eval runner:

- You have access to the target environment (Production, Dev, or Preview)
- The PR under test is deployed to the target environment and is the active build
- You have a fresh browser session — no leftover sessions from a prior run

### Running the Eval

1. Open the eval runner: `agent-evals/eval-runner/eval_runner.html` in your browser.
2. Select the environment from the dropdown (Production / Dev / Preview).
3. Record the PR number, build date, model version, and environment in the header before scoring begins.
4. Select TC1. Work through each step in order. For each step:
   - Read the 'Behavior Being Tested' and 'Expected Output' columns.
   - Trigger the agent action (use the copy button to paste prompts).
   - Observe the agent's response and compare it to the expected output.
   - Mark PASS, FAIL, SKIP. Add a note for any non-PASS result. Add other observations to the notes.
5. Open up a second browser and start TC2 if you want to speed up the process.
6. After both TCs are complete, export the results using the 'Export CSV' button.

### Scoring Rules

Apply the following scoring definitions consistently across all runs:

| Value | Definition | Example |
|-------|-----------|---------|
| **PASS** | The agent's observed behavior matches the expected output for this step. | Agent correctly asks for SSN instead of assuming a value. |
| **FAIL** | The agent's behavior does not match the expected output. Describe the deviation in Notes. | Agent assumed homelessness = No without asking (Step 34). |
| **SKIP** | Step is not applicable in this environment or test case variant. Excluded from pass-rate denominator. | Step 5 (Carlos DOB) — always SKIP in Production because Carlos is not linked to Rosa's A360 in prod. |
| **Unscored** | Step attempted but result was ambiguous, partially complete, or interrupted. Excluded from pass-rate denominator. Add context in Notes. | Session terminated early due to Take Control crash. |

**Pass rate formula:** `PASS ÷ (PASS + FAIL)`. Steps scored SKIP or Unscored are excluded from both the numerator and denominator.

> **Note:** A step's rubric determines what is being tested. If the agent does something unexpected but still satisfies the expected output, score PASS and describe the deviation in Notes. If it satisfies the output but also does something harmful (e.g., invents a value for a field not covered by this step), score PASS and add a note flagging the additional concern separately.

---

## Generating the Comparison Report

For every Preview / PR eval run, produce a report comparing the PR against the current Production baseline and the most recent Dev run (if one exists).

### Report Generator

The comparison report is a Word document (`.docx`) generated by a Node.js script (`generate_prNNN.js`). The script lives in `agent-evals/report-generator/` and is versioned with the project.

- Open or duplicate the most recent `generate_prNNN.js` script.
- Update the `RUNS` array at the top with the three runs being compared (Baseline, Dev, PR). Fill in: `label`, `model`, `env`, `date`, `passRate`, `passCount`, `rateFill`, `rateColor`.
- Update `CATEGORIES` with per-category pass rates for all three runs.
- Update `RUBRIC_NARRATIVE` with a headline, bullets for regressions, and a wins summary.
- Update `FAILURES` with the notable failures and their status across all three runs. Include any steps that scored PASS but show concerning behavior (mark as `WARN`).
- Run: `node generate_prNNN.js`
- Open the `.docx` in Word and verify the tables render correctly and all pass rates match the CSVs.
- Save to `agent-evals/report-generator/` and commit.

### Report Contents Checklist

Every comparison report must include:

- Summary table: model, environment, date, scored steps, pass rate
- Category breakdown table with pass rates per category across all runs and delta vs. baseline
- Rubric narrative: headline, what needs attention, what improved
- Improvements table: steps that moved from FAIL → PASS vs. the prior run
- Failures table: all notable failures and their status across all runs
- Source footnote: names all CSV files used, pass-rate formula, suite description

---

## Merge Readiness Thresholds

| Pass rate | Recommended action |
|-----------|-------------------|
| **≥95% — Green** | PR is a strong pass. Proceed to code review. |
| **89–94% — Yellow** | Meets baseline. Review category-level regressions before merging. Flag any new failures introduced by the PR. |
| **85–88% — Amber** | Borderline. Identify root cause of failures. PR may need iteration before merge. |
| **≤84% — Red** | Do not merge. Investigate and remediate before re-running eval. |

> **Note:** Pass rate thresholds are a starting point. A 95% run with a new SSN hallucination regression should not be merged without investigation. Use judgment when novel failure modes appear.

---

## Filing a Bug Ticket

File a Jira ticket when:

- A step that was passing in Production baseline begins failing in a PR or Dev run (new regression)
- Critical bugs: anything is hallucinated, session data leaks across users or sessions (cross-session contamination), session crash

**Ticket format — follow the existing ASP-XXX Jira style:**

- Title: `[Dev/Preview/Prod]` Short description of the failure
- Type: Bug, Priority: High (PII / session issues) or Medium (functional regressions)
- Steps to reproduce with exact input prompt and record ID
- Session ID link from the environment
- Expected vs. actual behavior
- Sprint assignment and relation to any predecessor tickets

---

## Known Persistent Issues

The following failures are tracked and have been present across multiple runs. Do not mark these as unexpected regressions in a new PR — document them as persistent and note whether the PR improved, worsened, or did not affect them.

| Step | Issue | Jira ticket |
|------|-------|-------------|
| **Step 4** | Auth checkbox (TC1 — WIC form): Agent asks caseworker to confirm instead of checking the authorization box automatically. | Open — no ticket |
| **Step 34** | Homelessness status (TC1 — BenefitsCal): Agent assumes 'No' despite system instruction to ask. | Open — no ticket |
| **Steps 40/57** | Verbosity (TC1 & TC2): Agent over-narrates steps and uses technical jargon in caseworker-facing messages. Stuck at 50% across all runs. | Open — no ticket |
| **Step 18 ⚠** | SSN hallucination (TC1 — WIC): Agent invents '123-45-6789' in the Conversation Checkpoint after correctly noting SSN was absent. Rubric scores PASS (tests ask behavior only). | **ASP-924 — High priority** |
| **Step 53** | BenefitsCal gap analysis (TC2): Agent asks for program and Authorizing Representative separately instead of a single gap summary. Inconsistent across runs. | Open — no ticket |

---

## Quick Reference

| Item | Detail |
|------|--------|
| GitHub repository | https://github.com/christinewilkes-commits/AI-Evaluations |
| Eval runner | `agent-evals/eval-runner/eval_runner.html` |
| Report generator scripts | `agent-evals/report-generator/generate_prNNN.js` |
| CSV archive | `agent-evals/report-generator/*.csv` |
| Test case count | 57 steps — TC1 (Rosa Flores) + TC2 (Carolina Delgado) |
| Pass rate formula | `PASS ÷ (PASS + FAIL)` — SKIPs and Unscored excluded |
| Target baseline (prod) | 89% — Run 1, May 4, 2026, claude-opus-4-7 |
| SSN hallucination ticket | ASP-924 — open, High priority |

---

*Nava PBC — Internal Document. Do not distribute outside the project team.*
