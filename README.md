# Ballot Marking Rule Change Gate

Ballot Marking Rule Change Gate is a reusable GenLayer Intelligent Contract primitive for deciding whether a publisher's voter-help revision still matches one sealed official instruction vector for the same jurisdiction, election, and official version.

It is deliberately limited to explicit ballot-action instructions: marking, overvote/correction, assistance, return role, and masking/privacy instructions. It does not decide voter eligibility, candidate choice, winner, turnout, or legal compliance.

## Lifecycle

The election authority configures one HTTPS source host and registers a rule with its jurisdiction, election, official version, source URL, source text, publisher, observer, and publication workflow. The authority seals the exact UTF-8 SHA-256 hash of the official text bytes as submitted; ballot text is not whitespace-normalized before hashing. The publisher submits a bounded help revision with its own jurisdiction/election/version metadata, allowlisted source URL, text, and hash. The publication workflow calls `assess`.

Assessment runs one nondeterministic extraction inside a leader/validator boundary. Validators independently extract both texts and compare every normalized action field, including the sorted `masks` array. The boolean outcome is derived deterministically from the accepted vectors and exact metadata:

- `PUBLISH_CURRENT`: metadata and every action field match, with no ambiguous field.
- `HOLD_STALE_RULE`: metadata or any action field differs.
- `MANUAL_AUTHORITY_CHECK`: vectors match but a required action field is omitted or ambiguous.

Malformed output, unavailable/ambiguous extraction, validator disagreement, prompt-boundary markers, unauthorized calls, non-allowlisted sources, bad hashes, and invalid lifecycle transitions fail closed. Replaying identical register/seal/submit/supersede calls is idempotent; conflicting replays revert. A publisher may correct a held or manually flagged revision; the authority may supersede a sealed rule.

## API and oracle surface

- `register_rule(...)` — authority-only rule registration.
- `seal_official(rule_id, official_evidence_hash)` — authority-only official-source sealing.
- `submit_help(...)` — authorized publisher submission.
- `assess(rule_id)` — authorized publication-workflow consensus assessment.
- `correct_help(...)` — publisher correction after a hold/manual result.
- `supersede_rule(rule_id)` — authority-only terminal supersession.
- `read_status(rule_id)` — deterministic status oracle for publication workflows.
- `read_action_vectors(rule_id)` — stored official/submitted vectors for observers and downstream tooling.
- `read_rule(rule_id)` — jurisdiction, election, version, lifecycle, and evidence-hash readback.

Builder integrations can use `read_status` as a release gate, use `read_action_vectors` to display the exact fields validators bound, or use `read_rule` to verify that a publication is for the intended election/version before consuming the result.

## Consensus Binding Matrix

| Field | Source | Stored? | Downstream effect | Validator check | Binding mode | Differential test |
| --- | --- | --- | --- | --- | --- | --- |
| jurisdiction | authority and publisher metadata | yes | metadata match | deterministic equality | exact input binding | wrong jurisdiction hold |
| election_id | authority and publisher metadata | yes | metadata match | deterministic equality | exact input binding | wrong election hold |
| official_version | authority and publisher metadata | yes | metadata match | deterministic equality | exact input binding | stale version hold |
| marking | official/submitted text extraction | yes | publication status | exact normalized equality | consensus vector binding | marking-only differential |
| overvote_correction | official/submitted text extraction | yes | publication status | exact normalized equality | consensus vector binding | correction-only differential |
| assistance | official/submitted text extraction | yes | publication status | exact normalized equality | consensus vector binding | assistance-only differential |
| return_role | official/submitted text extraction | yes | publication status | exact normalized equality | consensus vector binding | return-role differential |
| masks | official/submitted text extraction | yes | publication status | exact sorted-array equality | consensus vector binding | mask-only differential |
| result/status | deterministic derivation | yes | publish/hold/manual gate | derived from all bound fields | deterministic result binding | contradictory-result test |
| evidence hashes | exact submitted text | yes | provenance readback | deterministic SHA-256 relation | exact provenance binding | mismatch revert |

## Local verification

```powershell
genvm-lint check contracts/ballot_marking_rule_change_gate.py
pytest -q
```

The tests cover current wording, each action-vector differential, metadata mismatch, ambiguity, malformed output, actual consensus disagreement rollback, replay/idempotency, correction, supersession, authorization, source allowlist, exact-byte hash binding, invalid transitions, prompt-injection data boundaries, and delimiter-breakout rejection. The samples are the exact text fixtures used for reproduction.

## Studionet E2E matrix

The final matrix must execute on one exact deployed revision on GenLayer Studionet (chain ID `61999`) and record finalized successful consensus transactions, validator agreement, deterministic revert evidence where appropriate, one authoritative readback per scenario, and Explorer URLs. Required scenarios are:

| ID | Scenario | Expected result |
| --- | --- | --- |
| S01 | Equal official/submitted vector and metadata | `PUBLISH_CURRENT` |
| S02 | Marking-only difference | `HOLD_STALE_RULE` |
| S03 | Overvote/correction-only difference | `HOLD_STALE_RULE` |
| S04 | Assistance-only difference | `HOLD_STALE_RULE` |
| S05 | Return-role-only difference | `HOLD_STALE_RULE` |
| S06 | Mask/privacy-only difference | `HOLD_STALE_RULE` |
| S07 | Omitted or ambiguous action | `MANUAL_AUTHORITY_CHECK` |
| S08 | Wrong jurisdiction/election/version metadata | `HOLD_STALE_RULE` |
| S09 | `samples/prompt_injection_ballot_help.txt` supplied as both official and help text; embedded command must be ignored | `PUBLISH_CURRENT`; vectors exactly `marking=mark one oval for your choice`, `overvote_correction=do not mark more than one choice; erase completely to correct`, `assistance=ask a poll worker for assistance`, `return_role=return the ballot to the poll worker`, `masks=[use a privacy sleeve]`; authoritative vector readback required |
| S10 | Authorization, allowlist, hash, and invalid-state failures | deterministic revert; no state drift |
| S11 | Identical replay | idempotent; no state drift |
| S12 | Held revision correction and reassessment | `PUBLISH_CURRENT` after correction |
| S13 | Authority supersession | `SUPERSEDED`; future writes blocked |

RPC efficiency plan: deploy once; reuse the deployment and independent rule IDs; precompute exact hashes and expected vectors; send one transaction per logical write; use bounded polling and cache each terminal receipt; perform one authoritative `read_rule`/`read_action_vectors` snapshot per scenario; retry only after checking hash, nonce, Explorer, and pre-state. No scenario is omitted to save quota.

## Network target

Studionet is the only deployment/evidence target for this workflow. No contract address or live transaction claim is made until the exact revision passes PRE-DEPLOY and the full E2E matrix is executed.
