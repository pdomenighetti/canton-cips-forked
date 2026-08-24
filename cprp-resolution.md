Number: CIP-XXXX

Title: Canton Party Name Resolution Standard

Status: Draft

Author: Paolo Domenighetti (Freename), [Co-authors TBD]

Created: 2026-08-23

## Abstract

This CIP defines the resolution layer for the Canton Network: how applications resolve human-readable names to Canton Party IDs and how they retrieve associated metadata. It specifies the Fully Qualified Party Name (FQPN) addressing format, the generic Resolver Interface that any identity provider implements, the resolution algorithm, the display conventions for rendered names, the built-in `party-id` resolver, guidelines for writing CIPs for additional resolvers, and a minimum fee model for on-ledger name state.

This CIP aligns with the CNS 2.0 architecture proposed by Simon Meier (Digital Asset) and the Working Group direction converged over the May–July 2026 meetings. Advanced name verification (a formal T1–T4 trust framework, per-source verification policies) is deferred to future work. Per-source import mechanics (DNS, LEI/vLEI, ENS, cross-chain) are specified in a companion CIP.

## Motivation

Canton participants are identified by cryptographic Party IDs — opaque strings of the form `<prefix>::<68-character-namespace-hash>`. These strings are unusable in any human workflow. CNS 1.0 names (`goldmansachs.unverified.cns`) are self-registered without verification, so they convey no signal about which counterparty they identify.

The Identity and Metadata Working Group has identified three concrete gaps:

- P1: Trustworthy human-readable names — applications need to display and accept names that map to specific Canton parties.
- P2: Off-ledger API endpoint discovery — applications need to find administrative endpoints (for example, the token-admin API for a CIP-56 instrument) keyed on a party identifier. A separate CIP-56 amendment (forthcoming from Digital Asset) will specify endpoint discovery on top of the CN Credentials Standard.
- P3: Self-published profile information — parties need to publish profile data that displays uniformly across applications, as specified in the Party Profile Credentials CIP (PixelPlex).

This CIP addresses the resolution machinery underlying all three: a uniform addressing model, a pluggable resolver interface, and a shared display convention.

## Specification

### 1. Terminology

- Party ID. A Canton ledger identity of the form `<prefix>::<namespace>` where the namespace is a 68-character hexadecimal hash of the party administrator's public key. Party IDs may contain additional `::` separators introduced by administrator delegation.
- Resolver. A software component that provides a mapping between names from an external or internal naming system and Canton Party IDs. Examples: `dns`, `lei`, `ens`, `cns`, `party-id`.
- Name Registry. On-ledger state, replicated in Canton via the CN Credentials Standard, that binds names within a resolver's scope to Party IDs.
- Registrar. A party approved to operate a name registry (or a subset of it) for a given resolver. Registrars are governance concepts, not part of the FQPN addressing format.
- FQPN. A Fully Qualified Party Name; see Section 2.
- Rendered name. The human-readable form of an FQPN shown in application UIs; see Section 6.

### 2. The Fully Qualified Party Name (FQPN)

#### 2.1 Format

A Fully Qualified Party Name is structured as three `:`-separated components:

```
<network>:<resolver>:<name>
```

| Component | Purpose | Example Values |
|-----------|---------|----------------|
| `network` | Prevents cross-environment confusion | `mainnet`, `testnet`, `devnet`, `localnet` |
| `resolver` | The naming system that backs this FQPN | `dns`, `lei`, `ens`, `cns`, `party-id` |
| `name` | A sequence of printable, non-whitespace ASCII characters, interpreted by the resolver | `blackrock.com`, `blackrock.canton`, `partyHint::hex-fingerprint` |

Examples:

- `mainnet:dns:blackrock.com`
- `mainnet:dns:tmmf.lloyds.com`
- `mainnet:cns:blackrock.canton`
- `mainnet:lei:784F5XWPLTWKTBV3E584`
- `mainnet:party-id:partyHint::1220abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000`

#### 2.2 Why free-form `<name>`

The `<name>` component is a sequence of printable, non-whitespace ASCII characters. It is opaque to the resolution layer and interpreted only by the resolver referenced by the `<resolver>` component. This design allows any external naming syntax to be embedded without escaping: DNS names (`blackrock.com`), CNS names (`blackrock.canton`), LEI codes (`784F5XWPLTWKTBV3E584`), and Canton Party IDs with their `::` separators all fit unchanged in the `<name>` slot.

The three-part format follows the direction proposed by Simon Meier in the June 25, 2026 CNS 2.0 next-steps presentation, after the June 18 Working Group discussion of whether a `registrar` component should appear in the FQPN. The direction: a registrar is a governance concept associated with a resolver, not a first-class addressing component. Adding a registrar slot would multiply icons in the UI, complicate the registrar lifecycle (what happens when a registrar changes for a namespace?), and offer little practical benefit to name discovery.

#### 2.3 Network Discrimination

The `network` component is mandatory. TestNet names MUST NOT be confusable with MainNet names. Applications SHOULD display the network when they support multiple networks.

#### 2.4 The `party-id` Resolver (Built-in)

The `party-id` resolver is built into this CIP and available on every network. It returns an FQPN whose `<name>` component is the Canton Party ID verbatim. This resolver guarantees that every Canton party always has at least one FQPN, even before any name has been registered by any other resolver. The `party-id` FQPN has no external trust anchor and no display name beyond the Party ID prefix.

Example: `mainnet:party-id:partyHint::1220abcd...`

### 3. The Resolver Interface

Every resolver — built-in or third-party — implements the following logical interface. HTTP and gRPC mappings are conventional; see Section 8.

| Method | Inputs | Output | Purpose |
|--------|--------|--------|---------|
| `resolve` | name | ResolutionResult | Forward lookup: name → Party ID + metadata |
| `reverseResolve` | party_id | ResolutionResult[] | Reverse lookup: Party ID → names known to this resolver |
| `resolveMulti` | name[] | ResolutionResult[] | Batched forward lookup |
| `changelog` | since-cursor | event stream | Subscribe to changes (additions, archivals, revocations) |

#### 3.1 ResolutionResult Schema

```
{
  "fqpn"          : "<network>:<resolver>:<name>",
  "party_id"      : "<canton-party-id>",
  "display_name"  : "<human-readable string>",
  "metadata"      : { /* claim keys and values */ },
  "valid_until"   : "<ISO-8601-timestamp>",
  "status"        : "OK" | "EXPIRED" | "NOT_FOUND"
}
```

The `display_name` is the resolver's canonical string for the name (typically the `<name>` component itself).

#### 3.2 Error Codes

| Code | Meaning |
|------|---------|
| 1000 | Name not found |
| 1001 | Resolver temporarily unavailable |
| 1002 | Malformed FQPN |
| 1003 | Network mismatch |

### 4. Name Resolution Algorithm

An application resolving a user-typed name follows a small, deterministic algorithm. The default is intentionally simple; multi-source composition is an optional advanced feature (Section 5).

#### 4.1 Basic Resolution

Given a user-typed string and an application's resolver configuration:

1. Match the string against each configured resolver's `resolve` in the order listed.
2. Return the first successful result.
3. If no resolver succeeds, return `NOT_FOUND`.

An application's resolver configuration is an ordered list with optional per-resolver ignore rules (regexes on the `<name>` component that cause the resolver to be skipped for matching names). Ignore rules address the case where a resolver serves a name space overlapping another and the application wants a specific delegation.

#### 4.2 Preferred-Name Resolution (Reverse Direction)

Given a Party ID, an application computes the preferred FQPN to display, following the algorithm from the CNS 2.0 next-steps presentation (June 25, 2026):

1. Retrieve the party's preferred-name credential from the DSO Credentials Registry: issuer = holder = the party, claim `cip-cns/preferred-name` (claim key per the CNS 2.0 CIP).
2. Interpret the claim value as an FQPN and resolve it forward per Section 4.1, verifying that it resolves back to the same Party ID. This round-trip verification prevents a party from claiming a preferred name it does not actually hold.
3. If verified, display that name.
4. If no preferred-name credential exists or verification fails, perform full reverse resolution: call `reverseResolve` on each configured resolver, collect all returned FQPNs, and apply the application's preference rule to select one. The default preference is lexicographic sort by (app-priority override, user-priority claim, app-priority default), per the configuration strawman in the same presentation.

The algorithm extends naturally to multiple preferred names by resolving all of them and applying the preference rule among the verified candidates.

### 5. Optional Advanced Feature — Multi-Source Composition

Applications with institutional audit or reverse-resolution needs MAY opt into multi-source composition, which aggregates results from multiple resolvers rather than short-circuiting on the first match.

Composition is advisory, not authoritative. The primary resolution flow of Section 4 returns a single canonical answer per name from the first matching resolver. Composition is a tool for use cases that need cross-source evidence: compliance audit trails (which resolver contributed which claim), cross-source corroboration for high-value settlement, and reverse resolution across an institutional footprint (which of my counterparty's known names should I display in this context).

Composition records per-claim provenance (`claim_sources`) tracking which resolver contributed each metadata value. When results agree on Party ID, metadata is merged with source attribution. When results disagree on Party ID, composition returns all candidates with their sources; composition does not itself pick a winner. The application decides how to present the disambiguation.

Composition is not the default resolution path. The simple resolution algorithm of Section 4 covers UI use cases, aligned with Leonid Rozenberg's observation (June 25, 2026) that name-selection UI in the wild is always context-scoped and never universally cross-source.

### 6. Display Conventions

Rendered names follow a shared convention: a source icon followed by a human-readable ASCII string.

```
<icon> <display-name>
```

The icon identifies the source of the name and is computed from the FQPN:

- For external-system resolvers (`dns`, `lei`, `ens`) and the built-in `party-id` resolver, there is one standardized icon per resolver.
- For the `cns` resolver, the icon is the logo of the TLD under which the name is registered. Registrars self-publish their desired logo for their TLD, and the logo is registered via a DSO-issued credential (`cip-cns/tld-logo-url`, per the CNS 2.0 CIP). At render time, the application retrieves the logo URL from the DSO Credentials Registry and displays `<img src=logoUrl> <display-name>`.

This keeps the icon set bounded and governable: external resolvers contribute a small fixed set of standardized icons, and `cns` logos are bounded by the set of DSO-registered TLDs rather than by the number of registrar organizations.

The `<display-name>` is the resolver's canonical form (typically `blackrock.com`, `blackrock.canton`, or a truncated Party ID prefix).

#### 6.1 Asset Names

Assets are identified by the Canton token standard as `InstrumentId{admin: Party, id: Text}`. The rendered form for an asset combines the admin's rendered name with the asset symbol:

```
<asset-symbol> by <admin-rendered-name>
```

Example: `CC by <cns icon> dso.cns`, `BUIDL by <dns icon> blackrock.com`.

The asset symbol is obtained by:
1. Resolving the admin Party ID to its preferred FQPN using this CIP.
2. Retrieving the admin's off-ledger token-standard API endpoint via the CN Credentials Registry (per the forthcoming CIP-56 amendment).
3. Querying that API at the token-standard endpoint `/registry/metadata/v1/instruments/{id}` to obtain the symbol published by the admin.

Assets do not receive their own FQPNs; they are addressed by admin + id per the token standard, and rendered via this convention.

#### 6.2 Three Rendering Contexts

| Context | Where used | Content |
|---------|------------|---------|
| Inline | Transaction lists, counterparty fields | `<icon> <display-name>` |
| Profile summary | Hover tooltip, popover | Rendered name + resolver + published profile claims |
| Full profile | Explorer profile page | Complete credential set, resolver history, changelog |

The full profile MAY be hosted by any explorer, not exclusively the Canton Foundation's Scan.

### 7. Trust and Verification

This CIP intentionally does not define a formal trust framework. The initial trust model is minimal:

- The DSO party issues credentials naming approved TLD registry admins and approved registrars per TLD (see the CNS 2.0 CIP led by Digital Asset).
- Each resolver's credibility follows from these DSO-issued credentials plus the resolver's own operational reputation.
- Applications trust resolvers by including or excluding them in their resolver configuration.

An advanced verification framework (four-tier T1–T4 model, per-source verification policies, application-configurable trust thresholds) is documented as future work in a companion draft (`cprp-verification.md`, retained as reference material). That framework is not part of the initial CIP set and is expected to be pursued once the resolution and import CIPs are stable.

### 8. Off-Ledger Resolution Service API and Availability

#### 8.1 API

The Resolver Interface (Section 3) is exposed by convention as an HTTPS + gRPC service.

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/resolve` | POST | Single forward lookup |
| `/v1/resolve/batch` | POST | Batched forward lookup |
| `/v1/resolve/reverse` | POST | Reverse lookup |
| `/v1/changelog` | GET (SSE) | Subscribe to changes |

The service is stateless; all state derives from on-ledger credentials plus externally cached data. Baseline footprint: 2 vCPU, 4 GB RAM per replica. No modification to SV nodes or to Scan is required.

#### 8.2 Availability

A naming layer materially less available than Canton mainnet is of limited value. Resolver implementations SHOULD target availability equal to or better than the Canton mainnet SLA they serve, using standard techniques: multiple stateless replicas across independent availability zones, health-checked load balancing, and read-through caching from on-ledger credential state.

When a resolver is unreachable, an application's resolver configuration MAY fall back to the next resolver in its list; the application MUST NOT silently substitute a lower-trust resolver for a higher-trust one without explicit user or configuration consent. Applications SHOULD surface resolver-unavailable conditions to the user rather than presenting a stale-cache result as fresh. The `party-id` built-in resolver (Section 2.4) is always available as a last-resort fallback that returns the Party ID itself as its FQPN, since it requires no external state.

The off-ledger placement is a deliberate design choice to limit ACS growth on the DSO party and to keep the Scan API surface narrow. The tradeoff is that the resolution layer's availability depends on resolver operator infrastructure rather than being an inherent property of the ledger. Applications with settlement-critical dependencies on name resolution SHOULD provision or contract for resolver infrastructure at the same availability standard as their ledger connectivity.

### 9. On-Ledger Representation

Name registrations are encoded as standard CN Credentials, using the credential registry defined by the CN Credentials Standard CIP (Digital Asset). This CIP does not introduce custom Daml templates.

A name-registration credential has publisher = the registrar, subject = the registered Canton party, holder = the registered Canton party. Claims include:

- `cprp/fqpn` — the canonical FQPN this credential certifies.
- `cprp/network` — the network.
- `cprp/valid-until` — expiry.
- Additional claim keys defined per resolver.

The `cprp/` claim-key prefix is a working prefix; it will be renamed to the `cip-<nr>/` form once this CIP receives its number, consistent with the claim-key convention used by the CNS 2.0 CIP (`cip-cns/*`) and the Party Profile Credentials CIP.

Import model. Names imported from external systems (DNS, LEI, ENS, and future importers) are materialized on-ledger as credentials so that Daml code can reference them. This follows the direction confirmed by Simon Meier in the WG meeting of July 9, 2026: "Import model is the only one that will allow records to be referenced by Daml code." Bridge-style references to external systems without on-ledger materialization are acceptable only for private or API-only resolvers where no Daml logic touches the name.

### 10. Fees

Name registration on-ledger consumes state and warrants a minimum fee model to prevent unbounded growth and to fund registrar operations. This CIP specifies the shape of the fee model, following the direction confirmed by Simon Meier at the WG meeting of July 9, 2026:

- Fees are paid per registered name.
- Read traffic (resolution queries) is free.
- Write traffic (registrations, updates, renewals) is paid via a fee.

The recommended implementation reuses the extended-duration credential mechanism introduced by the CN Credentials Standard CIP (proposed default fee of $1/year, first 90 days free, paid by CC burn requiring authorization from the funds owner and the issuer or holder). Registrars MAY charge above the minimum for higher-value names; this pricing latitude is one mechanism for reducing name squatting. Full tokenomics is out of scope for this CIP and expected to be addressed in a future governance CIP.

### 11. Guidelines for Writing CIPs for Additional Resolvers

Any project may propose a new resolver via its own CIP. Such a CIP MUST specify:

- The resolver's `<resolver>` identifier as it appears in FQPNs.
- The `<name>` syntax accepted by this resolver.
- The naming authority and governance under which registrars operate.
- The verification procedure by which the resolver establishes name-to-Party-ID bindings.
- The credential encoding used to publish on-ledger name registrations.
- The re-verification cadence, if the underlying authority state can change externally.
- The rendered-name convention if it differs from the `<name>` component verbatim.

Companion CIPs include the CNS 2.0 CIP (Digital Asset), the `.canton` CIP (Axymos, PR #209), and the Imported Names CIP (Freename, companion document) which specifies the DNS-verified name importer as a blueprint for further importers.

### 12. The `cns` Resolver and Private Registries

The `cns` resolver represents the Canton Name Service system defined by the CNS 2.0 CIP led by Digital Asset. Within CPRP, `cns` FQPNs are treated as one resolver among many: composable in resolution results and subject to the display conventions of Section 6, rendered with the per-TLD logo registered via the DSO Credentials Registry.

Allocation, uniqueness, and governance of the `.canton` name space and other `cns` name spaces are defined by the CNS 2.0 CIP and the `.canton` CIP (Axymos, PR #209), not by this CIP.

Private registries. Applications and institutions MAY operate private resolvers that expose the interface of Section 3 only to approved counterparties. Simon Meier confirmed at the WG meeting of July 9, 2026 that private registries will be supported (though they are not a priority for the MVP): the same API surface serves both public and private resolvers. Private resolvers publish the same FQPN shape and follow the same display conventions; access control is enforced at the credential-fetch and API-authorization layers, not at the resolution-protocol layer.

### 13. Backward Compatibility

This CIP is additive. Existing CNS 1.0 names (`name.unverified.cns`) are preserved unchanged; they resolve under the `cns` resolver with their existing display form. Party IDs continue to identify parties as they always have; the `party-id` resolver simply exposes them as FQPNs. Applications that do not adopt this CIP continue to work with raw Party IDs or with the `DsoAnsResolver` directly.

## Rationale

### Why three-part FQPN

The three-part FQPN follows the direction set in the June 18–25 Working Group discussions, in which the fourth `registrar` component was judged to solve no problem that could not be solved otherwise. Registrar identity is a governance concept associated with a resolver, best addressed by DSO-issued credentials naming approved registrars for each resolver. Embedding registrar in the FQPN forced per-registrar iconography, introduced registrar-lifecycle complications (name-value transfer between registrars), and made the addressing format do work better done by the credential layer.

### Why free-form `<name>`

Every external naming system has its own syntax — DNS uses dots and reverses order, LEI uses fixed-length alphanumeric codes, Party IDs use `::` separators. Trying to normalize these into a shared syntax either loses information or requires escaping conventions that undermine the format's readability. Making `<name>` a printable-ASCII-except-whitespace string cedes syntactic responsibility to each resolver, which is where domain knowledge about the syntax lives anyway.

### Why import model for on-ledger names

Simon Meier's answer at the July 9 WG meeting is definitive: Daml code that references a name needs the name to be on-ledger state, not a remote reference. Bridge/reference models are viable for read-only, private, or API-only cases where no smart contract depends on the name, but the general case for institutional Canton workflows requires import.

### Why simple resolution first, composition optional

Leonid Rozenberg observed at the June 25 meeting that in the wild — GitHub, Gmail, Outlook — name-selection UIs are always context-scoped and never universally cross-source. Making the simple first-match algorithm the default and multi-source composition an opt-in advisory feature respects that observation while preserving composition as a genuine tool for the audit and reverse-resolution cases that need it.

### Why this icon model

The June 18 WG discussion raised the concern that per-registrar iconography could produce an unbounded long tail of little-recognized glyphs. The model in Simon Meier's June 25 presentation addresses this by bounding the icon set at two levels: external resolvers (`dns`, `lei`, `ens`) contribute one standardized icon each, and `cns` logos are per-TLD — bounded by the set of DSO-registered TLDs, whose registration is itself a governance act. A registrar's logo therefore exists only where the DSO has registered the TLD it serves, keeping the visual vocabulary governable while still letting users recognize the source of a Canton-native name.

### Why the fee shape

Simon Meier committed the WG to a minimum fee model at the July 9 meeting: per-name fees, free read traffic, paid write traffic. Specifying this shape in the resolution CIP prevents free-riding on shared registrar infrastructure without over-specifying tokenomics that belong in a future governance CIP.

## Companion Documents

- CNS 2.0 CIP (Digital Asset) — the `cns` resolver and its associated registry infrastructure.
- Party Profile Credentials CIP (PixelPlex, PR #169) — profile claim keys used by rendering.
- CN Credentials Standard CIP (Digital Asset, PR #204) — the credential registry all resolvers publish into.
- CIP-56 amendment for off-ledger API discovery (forthcoming from Digital Asset) — endpoint discovery based on CN Credentials.
- Imported Names CIP (Freename, companion) — the DNS-verified name importer as a blueprint for further external-name importers.
- `.canton` CIP (Axymos, PR #209) — governance and allocation for the `.canton` name space.
- `cprp-verification.md` (reference / future work) — the advanced trust framework retained as a reference for future WG consideration.
