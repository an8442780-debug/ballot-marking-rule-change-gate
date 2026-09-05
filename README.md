# Ballot Marking Rule Change Gate

An auditable GenLayer Intelligent Contract that decides whether a publisher's voter-help revision still matches a sealed official instruction vector for the same jurisdiction, election, and official version.

## Live Deployment

- Network: GenLayer Studionet, chain ID `61999`.
- Contract: `0x1751Efa29Aaa0E4BaDF3a3f0e5d4E69a31878d10` ([Studio Explorer](https://explorer-studio.genlayer.com/address/0x1751Efa29Aaa0E4BaDF3a3f0e5d4E69a31878d10)).
- Deployer: `0xeF5D2119416A2f5afa35dCFA209766EFC1BE5902`.
- Deployment: [`0x5f960598a18c887b5b811dce54ec42ecb61a2f54a9c92ac2564cf094d38fcc60`](https://explorer-studio.genlayer.com/tx/0x5f960598a18c887b5b811dce54ec42ecb61a2f54a9c92ac2564cf094d38fcc60).
- Exact deployed source SHA-256: `19A5925E8C281585D0E8F95DBFC63D0EB7AAD5D1E19431EDD4CC7F9A81411575`.

| Evidence | Result |
| --- | --- |
| Fresh equal-vector S01b assess [`0x1a6f76322c6e4331105f0748096642e65368c21bf5fc29882f447394fe643f02`](https://explorer-studio.genlayer.com/tx/0x1a6f76322c6e4331105f0748096642e65368c21bf5fc29882f447394fe643f02) | `FINALIZED / SUCCESS`, `3 agree / 2 idle`, authoritative `PUBLISH_CURRENT` readback |
| Fresh assistance-difference S04b assess [`0x8d8ed9e50453db83e7f543112b968e5a88232d81494a98ed69398b612d8e8a8c`](https://explorer-studio.genlayer.com/tx/0x8d8ed9e50453db83e7f543112b968e5a88232d81494a98ed69398b612d8e8a8c) | `FINALIZED / SUCCESS`, `3 agree / 2 idle`, authoritative `HOLD_STALE_RULE` readback |
| Invalid-hash S10 [`0x16171e104400cf1c3e95214806377a908c492f7367a9e04959c29ebbb5553009`](https://explorer-studio.genlayer.com/tx/0x16171e104400cf1c3e95214806377a908c492f7367a9e04959c29ebbb5553009) | Deterministic finalized error; pre/post rule snapshots were unchanged |

The complete scenario record, including exact inputs, historical failure isolation, hashes, readbacks, and Explorer links, is in [verification/e2e-matrix.md](verification/e2e-matrix.md).

## Problem and Why GenLayer

Ballot-help revisions can silently change a voter action such as how to mark a choice, correct an overvote, request assistance, return a ballot, or protect privacy. This contract stores one sealed official text and compares a bounded publisher revision against the official action vector before publication.

GenLayer is useful here because validators independently extract meaning from source text and bind their agreement to the consequential status transition. A conventional backend is sufficient when the input is already structured, trusted, and deterministically validated; GenLayer is not needed for ordinary CRUD, private editorial workflows, or legal interpretation.

## How It Works

1. An authority registers a rule and seals the exact UTF-8 SHA-256 of the official text.
2. An authorized publisher submits bounded help text, metadata, source URL, and its exact hash.
3. The workflow calls `assess`; the leader and validators extract all five action fields from both texts.
4. Accepted vectors are compared deterministically with metadata and provenance hashes before state is written.
5. The oracle exposes `PUBLISH_CURRENT`, `HOLD_STALE_RULE`, or `MANUAL_AUTHORITY_CHECK`.

## State, Lifecycle, and Invariants

The lifecycle is `REGISTERED -> OFFICIAL_SEALED -> HELP_SUBMITTED -> ASSESSED`, with correction after a hold/manual result and terminal authority supersession. Stored fields include jurisdiction, election ID, official version, source and publisher metadata, exact evidence hashes, both normalized vectors, assessment state, status, and supersession state.

The contract fails closed on malformed or ambiguous extraction, validator disagreement, prompt-boundary markers, unauthorized calls, non-allowlisted HTTPS sources, bad hashes, and invalid transitions. Identical valid replays are idempotent; conflicting replays and post-supersession writes revert.

## Public API

Writes: `register_rule`, `seal_official`, `submit_help`, `assess`, `correct_help`, and `supersede_rule`.

Oracle views: `read_status` is the integrator release gate; `read_action_vectors` exposes the exact stored fields validators bound; `read_rule` returns lifecycle, metadata, and evidence-hash readback.

## Consensus and Security

Validators compare jurisdiction, election ID, official version, `marking`, `overvote_correction`, `assistance`, `return_role`, and sorted/deduplicated `masks`. Status is derived from the accepted complete vector, not from a model-proposed status string. User-controlled and fetched text is delimited as untrusted data; embedded commands are data and cannot alter the extraction instructions.

No eligibility, candidate, winner, turnout, or legal-compliance decision is made. Exact-byte hashing intentionally preserves provenance and does not whitespace-normalize submitted text.

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

## Local Verification

```powershell
genvm-lint check contracts/ballot_marking_rule_change_gate.py
pytest -q
```

The exact revision was verified with 29 passing tests and GenLayer lint/schema validation. Tests cover vector differentials, metadata mismatch, ambiguity, malformed output, disagreement rollback, replay/idempotency, correction, supersession, authorization, source allowlist, exact-byte hashing, invalid transitions, prompt injection, and delimiter breakout.

## Consensus Engineering Lessons

- Bind every action field, including arrays such as privacy masks, to the validator comparison.
- Derive the consequential status from accepted vectors and metadata instead of trusting an oracle status string.
- Treat exact source bytes as provenance; fixture newlines are part of the evidence hash.
- Delimit fetched and publisher-controlled text as hostile data and test delimiter-breakout attempts.
- A finalized transaction and an accepted consensus receipt still require authoritative state readback.

## Reusable Integrations

- A publication workflow can call `read_status` before making revised voter help visible.
- An observer UI can display `read_action_vectors` to show precisely which instructions were compared.
- A multi-election pipeline can bind each rule to its jurisdiction, election, and official version through `read_rule`.

## Limitations

This primitive only compares explicit voter-action instructions. It does not determine eligibility, candidate choice, winners, turnout, legal compliance, source truth, or whether an election authority's wording is legally sufficient. Ambiguity and validator disagreement intentionally stop publication and require authority review.

## Repository Structure

```text
contracts/ballot_marking_rule_change_gate.py
tests/test_ballot_marking_rule_change_gate.py
samples/official_ballot_help.txt
samples/submitted_ballot_help.txt
samples/prompt_injection_ballot_help.txt
verification/local-checks.md
verification/e2e-matrix.md
README.md
Notes.md
RESEARCH.md
requirements.txt
LICENSE
```

## License

MIT. See [LICENSE](LICENSE).
