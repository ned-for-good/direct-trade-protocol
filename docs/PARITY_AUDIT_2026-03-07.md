# DTP Spec ↔ Contract Parity Audit (2026-03-07)

## Summary
Contract compiles in containerized Rust check (`cargo check --target wasm32-unknown-unknown`) and current parity is strong for finance/freight schema wiring, with key execution gaps still explicitly placeholder.

## Enforced in contract now
- Finance terms validated (`net_days <= 60`, fee cap, escrow-only constraints).
- Freight terms validated (quote expiry ordering, allowance <= estimate).
- Landed-cost guardrail enforced on intent acceptance path when buyer pays freight.
- Finance + freight fields threaded through intent/listing/offer/contract structs.

## Documented but not fully enforced yet
- Project44 default quote source is documented; contract stores source enum but does not perform external quote fetch/verification.
- Freight booking at contract formation is represented via `booked_at_contract` flag, but no booking integration exists in contract runtime.

## Newly aligned during this hardening pass
- PACA due-date enforcement is now coded: when `finance.paca_covered=true`, contract validation requires `net_days <= 30`.

## Known placeholders (explicit TODO)
- Escrow lock is placeholder (`escrow-placeholder-*`).
- Settlement release tx is placeholder (`release-placeholder-*`).
- Matching reputation depth uses proxy and includes TODO for real reputation score lookup.

## Build validation artifact
- Toolchain: containerized `rust:latest` with PATH fix
- Command: `cargo check --target wasm32-unknown-unknown`
- Result: success (warnings only)
