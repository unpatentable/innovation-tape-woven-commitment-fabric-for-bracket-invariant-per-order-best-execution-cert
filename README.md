# Tape-Woven Commitment Fabric for Bracket-Invariant Per-Order Best Execution Certificates

> **Note:** This README is an ultra-condensed summary of the research reports published by Unpatentable.org. To view the full innovation documentation and explore the complete research, use the links at the bottom of this document.

**Domain:** [Social & Organizational Systems](https://unpatentable.org/innovation/?domain=social-organizational-systems)
**Subdomain:** [Algorithmic Trading](https://unpatentable.org/innovation/?domain=social-organizational-systems&subdomain=algorithmic-trading)

---

### Why We Invented This Technology
Roughly 60+ million US retail brokerage customers (and well over 100 million worldwide) route orders through brokers and wholesalers whose execution quality they cannot verify; academic studies (e.g., Schwarz, Barber et al., 'The Actual Retail Price of Equity Trades') found execution costs for identical orders vary several-fold across brokers, and pension and mutual fund savers bear similar hidden slippage, with plausible aggregate leakage in the low billions of dollars per year. Per person the harm is small (cents to dollars per trade) but it is persistent, regressive toward small and unsophisticated investors, and compounds over decades of retirement saving. Recent advances (practical zero-knowledge proof systems fast enough for large computations, 2023-2025 zkVM and folding-scheme progress, plus cheap nanosecond-grade timestamping and the SEC's expanded Rule 605 reporting) make it newly feasible to prove per-order execution quality cryptographically without exposing proprietary routing logic or other clients' flow; these figures are from my background knowledge and were not checked by a fresh search.

### The Innovation
## Abstract
The fabric interlaces two families of hash threads. A public "weft" is computed deterministically from the consolidated tape, so any SIP subscriber can recompute it. Private per-customer "warp" threads are kept by the broker. Each order crosses the weft twice: the customer's device signs the order over the latest weft root (the order cannot predate it), and the fill's public tape print later lands inside the weft (the execution cannot postdate it). A certificate of roughly 2 kB then compares the fill price with the NBBO extremum over that whole bracket, so the verdict does not depend on any broker timestamp. Verification is anisotropic: it runs freely along the customer's own thread and is blocked laterally toward other orders, routing and inventory.

## Background & Prior Work
SEC Rules 605/606 publish aggregate statistics, FINRA Rule 5310 reviews are internal, and the Consolidated Audit Trail is regulator-only. A customer receives a confirmation whose time is broker-asserted and cannot cheaply obtain the quote history to check it.

Cryptographic work covers the pieces separately:
- **Audit logs:** linked timestamping and Certificate Transparency logs (RFC 6962); hash-chain and Merkle audit trails of a firm's own trading events (VeritasChain posts).
- **Zero-knowledge audits:** episode-level compliance proofs for execution agents (arXiv 2510.04952).
- **On-chain markets:** batch-level price certificates (Bain Capital Crypto essay).
- **Attested data:** proofs attached to queried market data (Space and Time).
- **Time bracketing:** a public random beacon plus a trusted repository (arXiv cs/0308026).

These either log the broker's own view under the broker's own clock, need heavy provers, or assume on-chain venues. None found derives the freshness anchor and the price reference from one publicly recomputable tape commitment, or issues a verdict that is invariant to the execution timestamp.

## Competitive Landscape & Strategic Edge
Institutional transaction-cost-analysis vendors and broker execution-quality dashboards sell after-the-fact statistics that rest on trusted timestamps and licensed tick histories. Blockchain timestamping vendors prove only that a document existed by some time. Three blind spots are shared:
- **The broker's timestamp is assumed to need trust or proof.** This design makes the verdict hold for every instant in an externally fixed bracket.
- **Market state is assumed to need hiding with zero-knowledge machinery.** The NBBO is public; only lateral information (other orders, routes, inventory) is secret, so salted hashes suffice.
- **Proof delivery is assumed to need real-time data.** Delivering the proof late removes the data-cost problem.

The result is a hash-only proof that a watch can verify.

## Core Concept
**Weft (public tape threads).** A witness (any SIP subscriber) cuts the CTA/UTP quote and trade streams into epochs of Δ = 100 ms by SIP timestamp. Canonical record order is (SIP timestamp, feed ID, sequence number). For each symbol *s* active in epoch *e* it forms

L(s,e) = SHA-256(L(s,e′) ‖ e ‖ N₀ ‖ M_nbbo ‖ M_trade ‖ M_venue)

where:
- e′ is the symbol's previous active epoch;
- N₀ is the SIP-disseminated NBBO in force at epoch start;
- the M terms are Merkle roots over that epoch's NBBO-change records, trade prints and per-venue BBO updates.

The epoch root is R_e = SHA-256(R_{e−1} ‖ e ‖ t_last ‖ MerkleRoot{L(s,·)}). The tree is sorted by symbol and has at most 2¹⁴ leaves; t_last is the SIP timestamp of the epoch's final record. The construction is a pure function of the SIP stream, so independent witnesses, regulators or academics must obtain byte-identical roots, and equivocation is detected by an equality test. R_e depends on unpredictable market messages, so it also serves as a clock-free freshness beacon.

**Warp (private order threads) and anisotropic keying.** The broker holds master key K_B. Each customer device receives one thread key k_c = HMAC(K_B, "warp" ‖ id). The derivations are:
- event salts s_n = HMAC(k_c, n);
- leaves ℓ_n = SHA-256(s_n ‖ event);
- thread heads w_n = SHA-256(w_{n−1} ‖ ℓ_n).

Every second the broker places active thread heads in a fixed 2²⁰-slot sparse Merkle "selvage" tree, with slots derived from k_c and the epoch. Empty slots are padded with HMAC(K_B, "pad" ‖ e ‖ slot), so occupancy (flow rate) is hidden. The selvage root is countersigned by witnesses and logged. Routing events sit on a second thread keyed by HMAC(K_B, "route" ‖ id). That key is never given to the customer; it can be opened only with K_B, for example under a books-and-records request.

Verification conductivity is therefore directional:
- **Along the warp:** one 32-byte key opens the customer's whole history.
- **Laterally:** computationally zero.
- **Through the plane:** selective, as described next.

This copies two wearable structures. Woven e-textile buses use insulated conductive yarns that are joined only at chosen crossovers. Z-axis anisotropic conductive film conducts only where a particle is compressed between aligned pads.

**Two crossings per order (over/under).**
1. *Order over weft.* Before sending, the device's hardware-bound key (FIDO2/WebAuthn passkey, Secure Enclave or Android Keystore; ECDSA P-256) signs A = Sig(H(order terms ‖ nonce) ‖ i ‖ R_i), using the newest root it holds. A truncated H(A) rides in the existing client order ID field. The order therefore did not exist before t_lo = t_last(i). The broker rejects anchors more than two epochs stale and records the receipt leaf.
2. *Weft over fill.* Every execution must already be printed to the tape by a venue or a trade reporting facility. The fill leaf opens to (symbol, side, quantity, price p, execution timestamp t_x). The z-bond is the matching print (same symbol, price, size and participant timestamp) inside M_trade of some epoch. Its SIP timestamp is t_hi.

The bracket B = [t_lo, t_hi] is fixed by recomputable data rather than by the broker's clock.

**Certificate and grading.** For a marketable buy (sells are symmetric, using the bid), the prover reveals:
- the Merkle path of L(s,·) at the first and last bracket epochs (14 × 32 B each);
- the intermediate symbol leaves, chained through their back-pointers, where skipped epochs are provably inactive;
- every NBBO-change record in B, checked for completeness by recomputing M_nbbo;
- the print with its path.

The verifier computes NBO_min(B) and NBO(t_x), and checks t_x ∈ B. Grades:
- **A, bracket-invariant:** p ≤ NBO_min(B). The fill was at or better than the national best offer at every instant the order could have existed, with no timestamp trusted.
- **B, point-certified:** p ≤ NBO(t_x) only. The certificate reports timing slack S = p − NBO_min(B) and bracket width W = t_hi − t_lo.
- **Fail:** p > NBO(t_x), or t_x ∉ B.

Estimated size is 1.5–3 kB for typical retail brackets. Verification needs about 30–60 SHA-256 calls plus one or two signature checks, estimated at tens of milliseconds on a Cortex-M4-class wearable. For orders larger than the displayed NBBO size, an optional sweep proof opens M_venue to reconstruct per-venue BBOs and the size-weighted reference price.

**Commit live, reveal late.** Roots are published in real time and carry no usable quote content. The quote records inside certificates are released at least 15 minutes after the fill. From recall, the SIP plans treat data at that age as delayed data, exempt from per-user real-time fees; this needs confirmation with the plan administrators.

**Fail-wide property.** Every broker deviation can only widen B and so lower the grade. This covers relaying stale roots, holding the order, and reporting the print late. Latency becomes a visible, per-order number.

## Physics & Logical Feasibility
Soundness reduces to three standard assumptions: SHA-256 collision and preimage resistance, unforgeability of ECDSA or Ed25519, and the unpredictability of an epoch's market messages before they occur. No zero-knowledge circuits are needed, because the hidden items are simply never opened and salted commitments are computationally hiding.

Throughput is modest. Equity SIP traffic peaks at roughly 10⁵–10⁶ messages per second (estimate), while one SHA-extension core hashes several million short inputs per second (estimate). The 2²⁰-slot selvage tree costs about 2²¹ hashes per second.

Prerequisites and limits:
- The witness needs a SIP feed.
- The customer device needs a recent root, which the broker can relay since it cannot forge one.
- Bracket width is bounded below by mobile network latency and print-reporting latency, so some fills in fast-moving symbols will grade B. The certificate reports that outcome and does not hide it.

## Implementation Readiness & Key Constraints
**Months 0–3.** Publish the canonical weft specification. Build an open-source weft builder and test it on one purchased day of historical SIP-format data, including root reproducibility across two independent implementations.

**Months 3–6.** Build the client SDK (WebAuthn and platform keystores), the broker-side warp and selvage library, and a C verifier for microcontrollers.

**Months 6–12.** Run a paper-trading pilot with an API-first broker and two witnesses.

Estimated cash cost is under $10,000: data at most $2,000, cloud at most $1,500, test devices at most $1,000, and the remainder for an external cryptographic review. Engineering labor is assumed contributed.

Dependencies are a market-data legal opinion on hash roots and delayed reveal, exact handling of SIP corrections and cancels, and treatment of trade-reporting-facility latency (prints may lag by up to seconds).

## Utility & Purpose
Retail and small institutional investors get a per-order receipt they can check themselves or hand to any third party. It states that the fill was at or better than the NBBO for the entire interval in which the order could have been live, or else exactly how much timing uncertainty remains. Brokers and wholesalers gain a cheap, exportable defense in disputes and a way to compete on proven latency and price improvement without exposing routing logic, inventory or other clients' flow. Regulators gain tamper-evident, pre-committed routing records that can be opened under existing books-and-records powers. Immediate uses are retail apps, robo-advisers, small funds without a transaction-cost-analysis budget, and arbitration.

## Potential Applications & Impact
The weft is an enabling layer in its own right. A publicly recomputable commitment to the tape lets anyone build proofs about market state without holding tick archives. Possible uses:
- limit-order protection checks, showing that no print traded through a resting customer limit;
- verified execution-quality statistics, built by aggregating certificates rather than relying on self-reported Rule 605 tables;
- payment-for-order-flow research on certified samples;
- timestamp-dispute resolution.

The same weave transfers to options (OPRA), fixed income (TRACE prints) and European consolidated tapes as they come into operation.

Longer term, bracket-invariant grading changes the incentive. A broker's own delay degrades its own certificates, so the benefit of timing discretion shrinks and execution-quality competition moves from marketing claims to checkable artifacts.

## Market/Adoption Feasibility (If Applicable)
Marginal cost per certificate is a few kilobytes of storage and microseconds of hashing. A witness is one commodity server plus a SIP subscription. No venue, SIP or rule change is required: anchors travel in existing order-ID fields, prints already exist, and adoption is unilateral by any broker. Likely first adopters are API-first and reputation-sensitive brokers; investor-advocacy groups or universities can run witnesses. Barriers are market-data licensing interpretation, broker reluctance to expose latency, and customer comprehension, which a three-number display (grade, slack, width) is meant to address. All components (SHA-256, HMAC, passkeys, Merkle trees) are mature and royalty-free.

## Risk & Counterarguments
- **Reference price is narrow.** The NBBO omits hidden, midpoint and some odd-lot liquidity. A Grade A certificate proves "at or better than protected quotes," not global optimality or the absence of foregone price improvement.
- **Broker-authored apps control the signing surface.** A malicious app could delay signing. Mitigations are open, reproducible SDK builds, showing the anchor epoch at signing, or an independent companion verifier.
- **Double bonding.** One print could be bonded to two fills, and only a regulator holding K_B can detect this.
- **Wide brackets.** Prints may lag by up to seconds, which downgrades honest fills.
- **Licensing.** SIP plans might classify roots or delayed records as fee-liable.
- **Misreading.** Certificates could be oversold as full best-execution compliance, which also involves venue selection and regular review.

## Open Questions & Future Research Directions
- What is the empirical distribution of bracket widths and of Grade A share by symbol liquidity? This needs measurement and is not claimed here.
- Can a broker receipt countersigned by the customer device tighten the lower crossing without adding round-trip delay?
- Can public nullifiers detect double bonding without attributing prints to firms?
- How should the canonical weft handle SIP corrections, trading halts, the opening and closing crosses, and the new odd-lot and round-lot data?
- Should direct-feed wefts supplement the SIP weft?
- Would aggregated certificates satisfy or supplement Rule 605 reporting, and can succinct aggregation prove a month of grades in one object?
- How should the specification extend to options and non-US tapes?

## Novelty Threshold Justification
**Medium Novelty.** The primitives (hash chains, Merkle trees, beacons, salted commitments) are established. The closest prior art found:
- Time-bracketed authentication ([arXiv cs/0308026](https://arxiv.org/pdf/cs/0308026)) brackets documents with a random beacon and a trusted repository.
- VeritasChain-style logs ([dev.to](https://dev.to/veritaschain/building-tamper-evident-audit-trails-for-ai-trading-systems-a-deep-dive-into-cryptographic-logging-3bd2)) chain a firm's own events.
- [arXiv 2510.04952](https://arxiv.org/abs/2510.04952) proves episode-level compliance in zero knowledge.

Relative to these, three things are new:
1. A deterministic, publicly recomputable tape weft that serves at once as freshness beacon, upper time bound (through the fill's own print) and price reference.
2. Grading by the NBBO extremum over the bracket, which makes the verdict timestamp-invariant and fail-wide.
3. Three-axis anisotropic keying, combined with commit-live, reveal-delayed delivery.

Three web searches cannot exclude unpublished or patent-only disclosures.

## References & Prior Art Checks
From recall (no URL checked):
- Haber & Stornetta, "How to Time-Stamp a Digital Document" (1991).
- RFC 6962, Certificate Transparency.
- SEC Regulation NMS Rules 605, 606, 611, 613; FINRA Rule 5310.
- W3C WebAuthn / FIDO2.

Found by search:
- [Improvements to time bracketed authentication](https://arxiv.org/pdf/cs/0308026)
- [Safe and Compliant Cross-Market Trade Execution via Constrained RL and Zero-Knowledge Audits](https://arxiv.org/abs/2510.04952)
- [VeritasChain tamper-evident trading audit trails](https://dev.to/veritaschain/building-tamper-evident-audit-trails-for-ai-trading-systems-a-deep-dive-into-cryptographic-logging-3bd2)
- [Fairness by Design: Verifiable Execution in On-Chain Markets](https://stanford-jblp.pubpub.org/pub/fairness-by-design/release/1)
- [Bain Capital Crypto insights, "Public, Provable Prices"](https://baincapitalcrypto.com/insights/?cat=28)
- [Space and Time post](https://x.com/spaceandtime/status/2097731004496765078)
- [Linked timestamping](https://en.wikipedia.org/wiki/Linked_timestamping)
- [Trade Instruction Timing (TimeBinder)](https://timebinder.io/blog/trade-instruction-timing-determined-the-outcome-of-arbitration/)

**Prior art searched:**
1. "cryptographic proof of best execution per order zero-knowledge NBBO broker verifiable execution quality". Used: Stanford JBLP article, arXiv 2510.04952, Bain Capital Crypto essay, Space and Time post. No per-order NBBO proof system for traditional brokers was found.
2. "patent Merkle tree hash chain market data feed commitment tamper-evident timestamp order execution audit NBBO proof". Used: VeritasChain posts. No patents or market-data-feed commitments were returned.
3. "client-signed order anchored to public beacon hash proves order time bracket execution latency verifiable timestamp trading "best execution"". Used: arXiv cs/0308026, Linked timestamping, TimeBinder blog. No combined scheme was found.

---

### About the Reports
Each innovation on Unpatentable.org is published as a three-part open documentation set:
- **Innovation Report** – the master blueprint describing the first principals concept, mechanism, and rationale behind the invention.
- **Implementation Guide** – a step-by-step, practical manual detailing how to build or prototype the innovation. Includes estimated cost breakdown.
- **Industry Impact Report** – an analytical overview exploring the potential economic, industrial, and societal effects of the innovation.

Together, these documents form a complete public disclosure package — enabling researchers, developers, and policymakers to understand, replicate, and build upon each invention.

---

### Explore this Innovation
- [Innovation Report](https://unpatentable.org/innovation/tape-woven-commitment-fabric-for-bracket-invariant-per-order-best-execution-certificates)
- [Implementation Guide](https://unpatentable.org/innovation/implementation-guide-tape-woven-commitment-fabric-for-bracket-invariant-per-order-best-execution-certificates)
- [Industry Impact Report](https://unpatentable.org/innovation/industry-impact-report-tape-woven-commitment-fabric-for-bracket-invariant-per-order-best-execution-certificates)

### Download PDFs
- [Innovation Report (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Tape-Woven-Commitment-Fabric-for-Bracket-Invariant-Per-Order-Best-Execution-Certificates.pdf)
- [Implementation Guide (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Implementation-Guide-Tape-Woven-Commitment-Fabric-for-Bracket-Invariant-Per-Order-Best-Exec.pdf)
- [Industry Impact Report (PDF)](https://unpatentable.org/wp-content/uploads/2026/11/Industry-Impact-Report-Tape-Woven-Commitment-Fabric-for-Bracket-Invariant-Per-Order-Best-Ex.pdf)

---

**Timestamps:**
- [View Main Innovation Report on Arweave](https://arweave.net/_cuOHaYFfTga0ymsDL0C9l3Mn_UuwJtTCDdc8iLA-To)
- [View Implementation Guide on Arweave](https://arweave.net/qLZdX_1ITkVOp9rfD0TKeAcTLZL8cQIgG22tF_xY8q4)

**Suggested Citation:**
> Unpatentable.org (2026). *Tape-Woven Commitment Fabric for Bracket-Invariant Per-Order Best Execution Certificates.* Defensive Publication Series.
> https://unpatentable.org/innovation/tape-woven-commitment-fabric-for-bracket-invariant-per-order-best-execution-certificates

---

```json
{
  "domain": "Social & Organizational Systems",
  "subdomain": "Algorithmic Trading",
  "published": "2026-11-26",
  "arweave_main_tx": "_cuOHaYFfTga0ymsDL0C9l3Mn_UuwJtTCDdc8iLA-To",
  "license": "CC BY 4.0",
  "arweave_implementation_tx": "qLZdX_1ITkVOp9rfD0TKeAcTLZL8cQIgG22tF_xY8q4",
  "value_unlocked": "$39B"
}
```

Unpatentable.org is a public library of timestamped defensive publications —
open innovations released into the commons to prevent monopolization, democratize knowledge, and accelerate human progress.

License: CC BY 4.0
