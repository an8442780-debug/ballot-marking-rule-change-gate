# Local Verification

Local verification completed for the current prompt-schema correction candidate:

- `genvm-lint check contracts/ballot_marking_rule_change_gate.py`
- Exit code: `0`; lint passed (3 checks), contract validation passed, 9 methods detected (3 view, 6 write).
- `pytest -q`
- Exit code: `0`; `29 passed`.
- Contract SHA-256: `19A5925E8C281585D0E8F95DBFC63D0EB7AAD5D1E19431EDD4CC7F9A81411575`.
- Test SHA-256: `497D8372C02F4623708B147D71B73EC224A0B4167D00FBFC095D59A6CF10E504`.

Studionet integration and consensus transactions are not local checks and are not claimed here.
