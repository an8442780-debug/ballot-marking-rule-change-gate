# Local Verification

Local verification completed for the current prompt-schema correction candidate:

- `genvm-lint check contracts/ballot_marking_rule_change_gate.py`
- Exit code: `0`; lint passed (3 checks), contract validation passed, 9 methods detected (3 view, 6 write).
- `pytest -q`
- Exit code: `0`; `29 passed`.
- Contract SHA-256: `19A5925E8C281585D0E8F95DBFC63D0EB7AAD5D1E19431EDD4CC7F9A81411575`.
- Test SHA-256: `497D8372C02F4623708B147D71B73EC224A0B4167D00FBFC095D59A6CF10E504`.
- README SHA-256: `C03A54E7F576C6F837F251E4A1A9413684F56C239EAF02DD552921F1DCC95276`.
- Notes SHA-256: `CFAA9D6BEBAD5FDE49AA237AC3230046542F60FA831A683D31066F8EFF0E206A`.
- E2E matrix SHA-256: `41625D9F1DECE2686D391B4E0B1DAD02C77C8DF5035D77C812A772A864A62EE2`.
- LICENSE SHA-256: `B775B7FF9B4C0EA8BBB0E2499D9837469834DE81C85F64EEF3C51D1EBCEFF7B6`.

Studionet integration and consensus transactions are not local checks and are not claimed here.
