# DTP Finance Layer (v1)

Status: Draft implementation scaffold

DTP v1 finance is intentionally constrained for wholesale trade workflows.

The goal is to support payment terms without introducing marketplace complexity in the first release.

## Scope

DTP finance is currently scoped to **wholesale buyers and wholesale sellers**.

## v1 model

- Escrow remains the base settlement primitive.
- Buyers are not required to select a term; balances can be repaid at any time.
- Financing mode is either:
  - `escrow_only` (no external financer), or
  - `lp_pool` (single protocol LP lane in v1).
- Financing terms are recorded on-chain in intent/listing/offer/contract records.

## v1 LP finance policy defaults

For financed wholesale trades (`lp_pool`):

- LP yield target is fixed at 30% effective APR.
- Interest compounds daily.
- Buyer can prepay at any time with no prepayment penalty.
- Full payoff (principal + accrued interest + finance fees) is due by day 60.
- PACA-covered produce obligations are due by day 30.

This gives sellers immediate cashflow while keeping lender returns simple and legible. In all cases, maturity is deterministic: day 60 standard maximum, day 30 for PACA-covered produce.

## Why this shape

This keeps buyer UX simple, keeps seller economics predictable, and lets DTP collect repayment-quality data before opening lender routing and lender marketplaces.

## Market benchmark and value capture

Large wholesale distributors commonly offer early-pay terms like **2% discount if paid within 10 business days** (about 14 calendar days). That implies roughly **67.6% effective APR** for the payer-side capital (daily compounding assumption).

DTP v1 LP policy targets 30% effective APR for LPs. Even at that high investor yield, the financing drag is still cut by about half relative to a 67.6% benchmark while directing economics to protocol participants instead of legacy intermediaries.

## Default mitigation and underwriting controls (v1)

To protect LP capital while reputation data is still thin, financed access should be gated:

- buyer identity and business verification required before financing eligibility
- baseline creditworthiness check (trade references, payment history where available, and basic risk score)
- conservative initial exposure limits per buyer and per counterparty pair
- financing offered only after minimum protocol activity thresholds are met (or explicit manual approval)
- dynamic throttles: reduce or pause financed limits after late payments or dispute spikes
- short review cadence on LP health metrics (delinquency, default, recovery, concentration)

## Rollout posture

Start small and de-risk in production.

- launch with a modest LP size (not a large day-one pool)
- ramp limits and total pool capacity only after repayment performance validates assumptions
- treat early cycles as underwriting calibration, not growth maximization

## v2 direction (not in scope yet)

- buyer-selected funding partner
- multiple LP pools
- explicit pool risk tranching
- on-chain reputation scoring for payers and funders
- open funding marketplace
