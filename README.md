# Journal Entry Agent Extension: Posting Controls

An agent extension for the SAP **Journal Entry Agent** (`sap.agtjea:agent:journal-entry-agent:v1`, v1.0.1) that adds three guardrails before any journal entry is posted to the general ledger in SAP S/4HANA Cloud:

1. **Document header text is required.** The agent will not post an entry without one and asks the user to provide it.
2. **Debit/credit balance validation.** If total debits do not equal total credits, the agent blocks posting, states the exact difference, and asks the user to correct the line items.
3. **Mandatory pre-posting simulation.** The agent always runs the simulation and shows the result, including any errors or warnings, before asking the user to confirm posting.

The goal is to catch errors before they reach the ledger, reduce reversals and audit-trail noise, and make every posted entry audit-ready.

## Who it is for

General ledger accountants and finance staff who create and post journal entries (accruals, corrections, intercompany transfers) with the Journal Entry Agent.

## How it works

The extension is applied as custom instructions to the base agent. On every posting request, the agent checks the three guardrails in a fixed order:

| Step | Guardrail | If it fails |
|------|-----------|-------------|
| 1 | Header text is present and non-empty | Posting stops. The agent asks the user for a header text. No simulation or post call is made. |
| 2 | Total debits equal total credits | Posting stops. The agent reports debits, credits, and the difference. No simulation or post call is made. |
| 3 | Simulation runs and its result is shown | If the simulation fails, the agent reports the failure and does not post. Errors or warnings are shown before the confirmation question. |

The post API is called only after all three steps pass and the user explicitly confirms.

The agent also logs a milestone message for each guardrail (`M1`, `M2`, `M3`), marking it as achieved or missed, so the outcome can be observed.

### Example: unbalanced entry

```
The entry cannot be posted: debits total 5,000.00 EUR but credits total 4,500.00 EUR.
The difference is 500.00 EUR. Please correct the line items and try again.
```

## Repository contents

```
.
├── solution.yaml                              # Solution manifest
├── intent.md                                  # Business intent, fit-gap analysis, recommendation
├── product-requirements-document.md           # PRD: personas, goals, requirements REQ-01 to REQ-03
├── specification/
│   ├── specification.md                       # Top-level build checklist
│   └── journal-entry-posting-controls-agent-extension/
│       └── specification.md                   # Extension build spec and validation checklist
└── assets/
    └── journal-entry-posting-controls-agent-extension/
        ├── asset.yaml                         # Asset manifest (type: agent-extension)
        ├── extension.yaml                     # Custom instructions that enforce the guardrails
        └── aeval/                             # Agent evaluation suite
            ├── eval.yaml
            ├── config.yaml
            └── testcases/
                ├── 000_pre_setup.yaml
                ├── 01_Correctness_HappyPath_AllGuardrailsPass.yaml
                ├── 02_Correctness_MissingHeaderText_BlockedThenResolved.yaml
                ├── 03_Correctness_UnbalancedEntry_BlockedWithDifference.yaml
                ├── 04_Correctness_SimulationFailure_PostingBlocked.yaml
                ├── 05_Groundedness_NoInventedValues.yaml
                └── zzz_post_teardown.yaml
```

## Requirements

- SAP S/4HANA Cloud (Public or Private Edition) with General Ledger Accounting
- SAP Joule Studio, with access to the Journal Entry Agent (v1.0.1 or compatible)
- Permission to create and deploy agent extensions in the target tenant

## Installation

1. Download or clone this repository.
2. Open SAP Joule Studio and import the solution, or upload the `solution.yaml` package.
3. Attach the `journal-entry-posting-controls-agent-extension` asset to the Journal Entry Agent.
4. Run the evaluation suite in `assets/.../aeval/` to confirm the behavior in your tenant.
5. Deploy the extension to the environment where accountants work.

> Replace the steps above with the exact Joule Studio and deployment steps for your landscape.

## Testing

The `aeval` suite contains five scored test cases. Each one is mapped to the requirement it checks.

| # | Scenario | Guardrail(s) tested |
|---|----------|---------------------|
| 01 | All guardrails pass: happy path to posting | All three, positive |
| 02 | Missing header text: blocked, user provides it, then proceeds | REQ-03 |
| 03 | Unbalanced entry (off by 500 EUR): blocked with difference stated, corrected, then proceeds | REQ-01 |
| 04 | Header and balance OK, simulation fails: posting blocked | REQ-02 (failure path) |
| 05 | Groundedness and digression: no invented values; off-topic question handled | Groundedness, digression |

Coverage:

- **Requirements:** 5 response requirements, 0 tool requirements, and 5 rule requirements. Each limit is 10.
- **Test cases:** 5 of 15 allowed, covering correctness (4 scenarios) and groundedness plus digression (1 combined).
- **Traceability:** All 5 source requirements map to requirements, and all 5 map to tests, with parity in both directions.

The `000_pre_setup` and `zzz_post_teardown` files are optional setup and cleanup steps. They collect trace data but are not scored.

The `tool_requirements` list is intentionally empty because the extension adds no tools.

The business metric (100% header text coverage) needs a grounding run with real data before it can be scored.

Run configuration (`aeval/config.yaml`):

| Setting | Value |
|---------|-------|
| `lookback_duration` | 24h |
| `sampling_rate` | 0.5 |
| `cron_schedule` | `0 2 * * *` (daily at 02:00) |

> **Status:** The test definitions are in place, but no test reports have been generated yet. Run the suite in your tenant before relying on the extension.

## Scope and limitations

- **Instruction-based enforcement.** The guardrails are implemented through custom instructions. Hooks and MCP tools are disabled in this version (`enable_agent_extension_n8n_hooks` and `enable_agent_extension_mcp_server` are `false`), so the checks depend on the agent following its instructions. Test thoroughly before relying on them for production postings.
- **Journal Entry Agent only.** The extension does not apply to other agents.
- **No changes to posting rules.** Tolerances, account determination, and financial configuration in S/4HANA are not modified.
- **No approval workflow.** Multi-step authorization and reporting on posting quality are out of scope.

## Status

Version 1.0.0. The solution, specification, and extension files are complete, and the five evaluation scenarios are defined. Evaluation runs have not been executed yet. Intent fit is rated 95%.

## License

SAP

## Author

[Add name and contact here]
