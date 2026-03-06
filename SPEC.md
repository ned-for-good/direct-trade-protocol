# DTP Specification v0.1

**Direct Trade Protocol — Protocol Specification**

> Status: Draft | Version: 0.1 | Date: 2026-03-06

---

## 1. Overview

The Direct Trade Protocol (DTP) defines a standard set of message schemas, state machines, and settlement rules for coordinating the trade of physical goods between two parties without a broker or intermediary.

A DTP implementation consists of:
- **Messages** — structured data objects passed between parties
- **State machines** — valid states and transitions for intents, offers, contracts, and fulfillments
- **Settlement rules** — conditions under which escrowed funds are released or returned
- **Identity/credential references** — how party credentials are attached and verified

Any platform, agent, or application that conforms to this specification can interoperate with any other conforming implementation.

---

## 2. Core Concepts

### 2.1 Parties

A **Party** is any entity that participates in a DTP trade. Parties may be human-operated businesses or autonomous AI agents.

```json
{
  "party_id": "string",
  "account": "string",
  "business_name": "string",
  "business_type": "producer | distributor | retailer | cooperative | agent",
  "jurisdiction": "string",
  "certifications": ["CertificationRef"],
  "reputation": "ReputationRecord",
  "created_at": "ISO8601"
}
```

**Fields:**
- `party_id` — unique identifier (NEAR account name or DID)
- `account` — on-chain account address for settlement
- `business_type` — role in the supply chain
- `certifications` — array of `CertificationRef` objects (see 2.2)
- `reputation` — on-chain reputation record derived from completed trades (see 2.3)

### 2.2 CertificationRef

A certification claim is never self-asserted. Every certification must carry a reference to the issuing authority and be independently verifiable.

```json
{
  "cert_id": "string",
  "type": "string",
  "issuer": "string",
  "issuer_url": "string",
  "issued_at": "ISO8601",
  "expires_at": "ISO8601",
  "verification_url": "string",
  "status": "active | expired | revoked"
}
```

**Common certification types (food domain):**
- `USDA_ORGANIC` — USDA organic certification
- `FSMA_COMPLIANT` — FDA Food Safety Modernization Act compliance
- `GAP` — Good Agricultural Practices
- `FAIR_TRADE` — Fair Trade certified
- `HACCP` — Hazard Analysis Critical Control Points
- `NON_GMO` — Non-GMO Project verified

### 2.3 ReputationRecord

Reputation is built on-chain from completed trades. It cannot be manually set or imported.

```json
{
  "party_id": "string",
  "trades_completed": "integer",
  "trades_disputed": "integer",
  "trades_settled_on_time": "integer",
  "average_delivery_accuracy": "float",
  "score": "float",
  "last_updated": "ISO8601"
}
```

`score` is computed as: `(trades_completed - trades_disputed) / trades_completed * delivery_accuracy_factor`. Formula is on-chain and immutable per version.

---

## 3. Message Types

### 3.1 TradeIntent

A **TradeIntent** is a public declaration by a buyer of what they want to purchase. It is the entry point to a DTP trade.

```json
{
  "intent_id": "string",
  "version": "string",
  "buyer": "PartyRef",
  "goods": "GoodsSpec",
  "delivery": "DeliverySpec",
  "payment": "PaymentSpec",
  "expires_at": "ISO8601",
  "status": "IntentStatus",
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**IntentStatus state machine:**
```
DRAFT → POSTED → MATCHED → CONTRACTED → FULFILLED → SETTLED
                         ↘ EXPIRED
                         ↘ CANCELLED
```

### 3.2 GoodsSpec

Describes the goods being requested or offered. Used in both TradeIntent and Offer.

```json
{
  "category": "string",
  "product_name": "string",
  "description": "string",
  "quantity": {
    "amount": "decimal",
    "unit": "string"
  },
  "quality": {
    "grade": "string",
    "specifications": ["string"]
  },
  "required_certifications": ["string"],
  "packaging": "string",
  "shelf_life_days": "integer | null"
}
```

**Quantity units:** `lb`, `kg`, `oz`, `ton`, `case`, `pallet`, `unit`

**Required certifications** are cert type strings (see 2.2). An offer that does not carry all required certifications is ineligible for matching.

### 3.3 DeliverySpec

```json
{
  "destination": {
    "address": "string",
    "city": "string",
    "state": "string",
    "zip": "string",
    "country": "string"
  },
  "window": {
    "earliest": "ISO8601",
    "latest": "ISO8601"
  },
  "method": "delivered | FOB_origin | third_party_logistics",
  "temperature_requirements": "ambient | refrigerated | frozen | null",
  "notes": "string | null"
}
```

### 3.4 PaymentSpec

```json
{
  "currency": "USDC",
  "price_per_unit": "decimal",
  "total_ceiling": "decimal",
  "escrow_required": true,
  "payment_on": "delivery_attestation | inspection_period_end"
}
```

`price_per_unit` is the **maximum** the buyer will pay (in a TradeIntent). The accepted offer price may be lower.

`escrow_required` is always `true` in v0.

---

### 3.5 Offer

An **Offer** is a seller's response to a posted TradeIntent.

```json
{
  "offer_id": "string",
  "version": "string",
  "intent_id": "string",
  "seller": "PartyRef",
  "goods": "GoodsSpec",
  "delivery": "DeliverySpec",
  "payment": {
    "currency": "USDC",
    "price_per_unit": "decimal",
    "total_price": "decimal"
  },
  "certifications": ["CertificationRef"],
  "expires_at": "ISO8601",
  "status": "OfferStatus",
  "created_at": "ISO8601"
}
```

**OfferStatus state machine:**
```
SUBMITTED → SHORTLISTED → ACCEPTED
          ↘ REJECTED
          ↘ EXPIRED
```

An offer must be for the same or greater quantity and must carry all certifications required by the TradeIntent to be eligible for matching.

---

### 3.6 Contract

A **Contract** is formed when a buyer accepts an offer. It is the binding record of the agreed trade terms and triggers escrow.

```json
{
  "contract_id": "string",
  "version": "string",
  "intent_id": "string",
  "offer_id": "string",
  "buyer": "PartyRef",
  "seller": "PartyRef",
  "goods": "GoodsSpec",
  "delivery": "DeliverySpec",
  "payment": {
    "currency": "USDC",
    "price_per_unit": "decimal",
    "total_value": "decimal",
    "escrow_ref": "string"
  },
  "dispute_window_hours": "integer",
  "arbitrator": "PartyRef | null",
  "status": "ContractStatus",
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**ContractStatus state machine:**
```
ACTIVE → IN_FULFILLMENT → DELIVERED → SETTLED
                        ↘ DISPUTED → RESOLVED_BUYER
                                   → RESOLVED_SELLER
       ↘ CANCELLED
```

`escrow_ref` is the on-chain transaction ID where buyer funds are locked.

`dispute_window_hours` — the period after delivery attestation during which a buyer may raise a dispute before automatic settlement. Default: 48 hours.

---

### 3.7 Fulfillment

A **Fulfillment** record is created when a seller initiates delivery confirmation.

```json
{
  "fulfillment_id": "string",
  "contract_id": "string",
  "delivered_at": "ISO8601",
  "quantity_delivered": {
    "amount": "decimal",
    "unit": "string"
  },
  "seller_attestation": "Attestation",
  "buyer_attestation": "Attestation | null",
  "status": "FulfillmentStatus"
}
```

**Attestation:**
```json
{
  "party_id": "string",
  "signed_at": "ISO8601",
  "signature": "string",
  "notes": "string | null"
}
```

**FulfillmentStatus:**
```
SELLER_ATTESTED → BUYER_ATTESTED → COMPLETE
               ↘ DISPUTED
```

Settlement does not trigger until `COMPLETE` or dispute resolution.

---

### 3.8 Settlement

```json
{
  "settlement_id": "string",
  "contract_id": "string",
  "fulfillment_id": "string",
  "gross_amount": "decimal",
  "deductions": [
    {
      "reason": "string",
      "amount": "decimal"
    }
  ],
  "net_amount": "decimal",
  "currency": "USDC",
  "escrow_release_tx": "string",
  "settled_at": "ISO8601"
}
```

`escrow_release_tx` is the on-chain transaction ID for payment release to seller.

Deductions (quantity shortfalls, quality disputes resolved in buyer's favor) reduce `net_amount` from `gross_amount`.

---

## 4. Matching

Matching is the process of pairing a TradeIntent with the best eligible Offer. In DTP, matching is **off-chain** — a solver (human or agent) scores and ranks offers, and the accepted match is committed on-chain.

This design keeps gas costs low and allows matching logic to evolve without contract upgrades.

### 4.1 Eligibility Rules

An offer is eligible for matching against an intent if and only if:
1. The offer quantity ≥ intent required quantity
2. The offer carries all certifications listed in `intent.goods.required_certifications`
3. The offer price ≤ intent payment ceiling
4. The offer delivery window overlaps with the intent delivery window
5. The offer has not expired and has status `SUBMITTED`

### 4.2 Scoring

Eligible offers are scored on four dimensions (equal weight in v0):

| Dimension | Higher is better |
|---|---|
| Price (vs ceiling) | Lower price → higher score |
| Delivery timing | Earlier delivery → higher score |
| Seller reputation | Higher reputation.score → higher score |
| Certification depth | More certs than required → higher score |

The top 3 eligible offers are shortlisted and presented to the buyer for final acceptance.

### 4.3 Solver Role

In v0, the solver is a human operator or a simple scoring script. The protocol does not mandate solver implementation — any conforming matching algorithm may be used. Future versions will define a decentralized solver network.

---

## 5. Settlement Rules

1. Settlement is triggered when a Fulfillment reaches `COMPLETE` status (both party attestations received).
2. If the buyer does not attest within `dispute_window_hours` of seller attestation, settlement is triggered automatically (presumed acceptance).
3. If the buyer raises a dispute during the dispute window, the contract enters `DISPUTED` state. The arbitrator (defined in the contract, or a default DTP arbitrator in v0) resolves the dispute.
4. Escrow is held until settlement or dispute resolution. It cannot be unilaterally withdrawn by either party.
5. All deductions must be agreed by both parties or ordered by the arbitrator.

---

## 6. Audit Trail

Every state transition in DTP emits an on-chain event. The event log is append-only and cannot be modified or deleted.

**Event schema:**
```json
{
  "event_id": "string",
  "event_type": "string",
  "entity_type": "intent | offer | contract | fulfillment | settlement",
  "entity_id": "string",
  "actor": "string",
  "timestamp": "ISO8601",
  "payload_hash": "string"
}
```

`payload_hash` is a SHA-256 hash of the full event payload, enabling independent verification.

---

## 7. Versioning

DTP is versioned. Every message carries a `version` field. Breaking changes increment the major version. Implementations must reject messages with incompatible versions.

Current version: `0.1`

---

## 8. Reference Implementation

The canonical reference implementation consists of:

- **`contracts/`** — NEAR smart contracts (Rust) implementing the settlement layer
- **`sdk/`** — TypeScript SDK providing typed clients for all DTP message types and state transitions

Implementors are not required to use the reference contracts or SDK. Any conforming implementation is valid.

---

## 9. Out of Scope (v0)

The following are explicitly out of scope for DTP v0 and may be addressed in future versions:

- Automated IoT/sensor-based delivery verification
- Decentralized solver/matching network
- Cross-chain settlement (v0 settles on NEAR only)
- Regulatory compliance automation (FSMA, etc.) — DTP carries certification references but does not validate them
- Native DTP token or governance mechanism

---

*DTP is open protocol infrastructure. It is not a product, a marketplace, or a company.*
