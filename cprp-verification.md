Number: CIP-YYYY

Title: Canton Party Identity Verification — Trust Framework (Future Work / Reference Draft)

Status: Draft (parked as future work — not part of the initial CIP set)

Author: Paolo Domenighetti (Freename), [Co-authors TBD]

Created: 2026-05-28 (retitled 2026-08-23 to reflect deferred status)

## Status Note

This document is retained as a reference draft. Per Simon Meier's CNS 2.0 Proposal for Next Steps (presented June 25, 2026 and confirmed at the WG meeting of July 2, 2026), an advanced verification framework — with a formal T1–T4 tier model, application-configurable verification policies, a trust evaluator, a featured-resolver registry, and revocation semantics — is future work. It is not part of the initial CIP set targeted for the end-of-August 2026 publication milestone.

The initial trust model for the resolution and imported-names CIPs is minimal: the DSO party issues credentials naming approved TLD registry admins and approved registrars per TLD (per the CNS 2.0 CIP led by Digital Asset), and applications trust resolvers by including or excluding them in their resolver configuration. That model is sufficient for the resolution mechanics, DNS import, and initial `.canton` allocation.

This draft is kept visible in the repository so that when the WG advances to per-source trust tiering — a conversation likely to return once the resolution, imports, and CNS 2.0 CIPs are stable — the design work here is already available as a starting point. The content below reflects the state as of May 28, 2026, and has not been maintained against subsequent architectural decisions (the FQPN registrar removal, the `:` separator, the DNS import model). When this draft is revived as an active CIP, those alignments will be required.

## Abstract

This CIP defines the trust framework that determines whether a resolved Canton party identity qualifies as "verified" — the condition for removing the `.unverified` prefix carried by self-registered CNS 1.0 names. It specifies a four-tier classification of credential issuers (T1–T4), an application-configurable verification policy schema, a trust evaluation algorithm that yields a `VERIFIED` / `PARTIAL` / `UNVERIFIED` / `COLLISION` / `ERROR` judgment, the featured-resolver registry governed by Super Validator (SV) vote that grants T3 issuer status, and the revocation semantics that bound how quickly trust changes propagate.

## Motivation

The Identity and Metadata Working Group's central framing question — "How do we remove the `.unverified` prefix?" — is fundamentally a trust question. CNS 1.0 names are first-come-first-serve with no ownership check, so a resolved name like `goldmansachs.unverified.cns` conveys no signal about whether it actually belongs to Goldman Sachs.

A workable answer must:

- Be uniform across applications, so that "verified" means the same thing in a block explorer, a settlement system, and a compliance tool.
- Be configurable per application, because different applications legitimately have different risk tolerances.
- Compose with multiple identity sources at once without locking the framework to any one.
- Be governance-light, requiring SV votes only at the boundary where authority is granted (featured resolvers) and not in normal-path operation.

## Specification (Reference — not maintained since May 28, 2026)

### 1. Trust Tiers

Every credential consumed by trust evaluation declares a tier via the `cprp/trust-anchor` claim. Tiers reflect the credential's authority source.

| Tier | Authority Source | Granted By | Weight Range | Example Issuers |
|------|------------------|------------|--------------|-----------------|
| T1 | DSO / SV Consensus | On-ledger SV vote | 1.0 | SV-verified DNS claims; DSO arbitration credentials |
| T2 | Regulated identity authority | External regulation (GLEIF, KYC registry, national ID) | 0.7–0.9 | vLEI issued under GLEIF; KYC-verified credentials from regulated providers |
| T3 | Featured resolver | SV governance vote (annual renewal) | 0.4–0.6 | Featured-resolver DNS verifications; featured-resolver ENS verifications; featured-resolver native registrations |
| T4 | Self-attested | None | 0.1–0.2 | Party-published profile claims; unverified LEI lookups; self-attested SWIFT BIC |

#### 1.1 Tier Issuance Rules

- T1 credentials are issued only by the DSO party as a result of on-ledger SV consensus.
- T2 credentials are issued by featured resolvers (T3) when the resolver acts as a faithful conduit for an external regulated authority's attestation. The T2 tier reflects the external authority's trust, not the resolver's.
- T3 credentials are issued by featured resolvers under their own authority for the specific verification methods they are featured for.
- T4 credentials are issued by any party, including a party publishing claims about itself.

### 2. Verification Policies

A verification policy is an application-specific JSON document declaring how that application converts a composed resolution result into a trust verdict.

```
{
  "verification_policy": {
    "minimum_tier"         : "T2",
    "minimum_resolvers"    : 1,
    "minimum_total_weight" : 0.7,
    "require_methods"      : ["dns", "vlei"],
    "collision_handling"   : "strict" | "permissive",
    "expiry_grace_period"  : "PT0S"
  }
}
```

Two reference policies:

`INSTITUTIONAL_DEFAULT` — settlement, custody, regulated trading:
```
{ "minimum_tier": "T2", "minimum_resolvers": 1,
  "minimum_total_weight": 0.7, "collision_handling": "strict" }
```

`PERMISSIVE_DEFAULT` — block explorers, consumer wallets, informational UIs:
```
{ "minimum_tier": "T4", "minimum_resolvers": 1,
  "minimum_total_weight": 0.1, "collision_handling": "permissive" }
```

### 3. Trust Evaluation Algorithm

Given a composed resolution result and a verification policy, the evaluator returns a verdict in `{ VERIFIED, PARTIAL, UNVERIFIED, COLLISION, ERROR }`:

1. If the result status is `COLLISION` and policy is `strict`, return `COLLISION`.
2. Enumerate credentials whose `cprp/valid-until` has not elapsed (with grace).
3. Compute cumulative weight from each credential's `cprp/trust-anchor`.
4. If cumulative weight ≥ minimum, at least one credential meets the minimum tier, distinct featured-resolver issuers ≥ minimum, and required methods all present, return `VERIFIED`.
5. If some credentials exist but thresholds not met, return `PARTIAL`.
6. If no credentials exist, return `UNVERIFIED`.
7. If any required input is malformed, return `ERROR`.

The evaluator does not alter resolution results; it attaches a trust verdict.

### 4. Featured Resolver Registry

Featured resolvers attain T3 issuer status through an on-ledger SV governance vote. Featured status is bound to a set of registrars and verification methods, renews annually, and is revocable.

```
publisher : <DSO-party>
subject   : <resolver-operator-party>
holder    : <resolver-operator-party>
claims    : {
  "cprp/featured-resolver"  : "true",
  "cprp/featured-since"     : "<ISO-8601-timestamp>",
  "cprp/featured-cip"       : "<CIP-number-that-approved>",
  "cprp/featured-registrars": "<comma-separated-registrar-list>",
  "cprp/featured-methods"   : "<comma-separated-method-list>",
  "cprp/featured-renewal"   : "<ISO-8601-timestamp>",
  "cprp/trust-anchor"       : "T1"
}
```

### 5. Revocation Semantics

| Event | Maximum Propagation Time |
|-------|-------------------------|
| Credential past `cprp/valid-until` | Immediate (client-side, on read) |
| Issuer-initiated revocation | ≤60 seconds via changelog |
| `ResolverFeaturedStatus` revocation | ≤60 seconds via changelog |
| DNS record removal (Phase-1 DNS credentials) | ≤7 days |
| vLEI status change at GLEIF | ≤24 hours |

## Rationale (Reference — not maintained)

The rationale for four tiers, app-driven policies, credential-declared trust anchors, and separation from resolution reflects the design as of May 28, 2026. When this draft is revived, the rationale will need updating against the CNS 2.0 architectural decisions (registrar-out-of-FQPN, import model, resolver-icons-only) that followed.

## Path Forward

This document remains in the repository as reference material. It is not part of the initial CIP set publication targeted for end-of-August 2026. If and when the WG advances to formal per-source trust tiering, this draft is available as a starting point that will need to be re-aligned to the resolution, imports, and CNS 2.0 CIPs.
