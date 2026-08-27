Number: CIP-XXXX

Title: Canton Party Name Resolution Standard

Status: Draft

Author: Paolo Domenighetti (Freename), [Co-authors TBD]

Created: 2026-08-27

## Abstract

This CIP standardizes four things and deliberately nothing more: how human-readable party names are represented on the Canton Network (the Fully Qualified Party Name format), how they are resolved to Canton Party IDs, how they are rendered in application UIs, and how additional name sources are integrated by defining new resolvers or extending existing ones.

It aligns with the CNS 2.0 architecture proposed by Simon Meier (Digital Asset) and the Working Group direction converged over the May–August 2026 meetings.

## Out of Scope

To preserve implementation freedom, this CIP does not specify: transport protocols, API endpoints, or wire schemas for resolver services; deployment topologies, footprints, or availability targets; fee or economic models (these belong to the CIPs of the registries that hold name state, e.g. CNS 2.0); multi-source result composition; advanced verification frameworks (parked as future work); or the allocation and governance of any specific name space (`.canton` is governed by its own CIP, PR #209; the `cns` registry infrastructure by the CNS 2.0 CIP).

## Motivation

Canton participants are identified by cryptographic Party IDs of the form `<prefix>::<68-character-namespace-hash>` — unusable in human workflows. CNS 1.0 names (`name.unverified.cns`) are self-registered without verification and convey no signal about which counterparty they identify. Applications need to display trustworthy human-readable names, accept them as input, and discover metadata keyed on a party — uniformly across the network. This CIP provides the shared representation, resolution behavior, and rendering conventions that make that possible, while leaving every implementation choice to implementers.

## Specification

### 1. Terminology

- Party ID. A Canton ledger identity of the form `<prefix>::<namespace>`, where the namespace is a 68-character hexadecimal hash of the party administrator's public key. Party IDs may contain additional `::` separators.
- Resolver. A component that maps names from some naming system to Canton Party IDs. Examples: `dns`, `lei`, `ens`, `cns`, `party-id`.
- Registrar. A party approved to allocate names within a resolver's naming system. Registrars are a governance concept defined by each resolver's CIP; they are not part of the addressing format.
- FQPN. A Fully Qualified Party Name; see Section 2.

### 2. Representation — the Fully Qualified Party Name

#### 2.1 Format

```
<network>:<resolver>:<name>
```

| Component | Purpose | Values |
|-----------|---------|--------|
| `network` | Prevents cross-environment confusion | `mainnet`, `testnet`, `devnet`, `localnet` |
| `resolver` | The naming system that backs this FQPN | `dns`, `lei`, `ens`, `cns`, `party-id`, others via their own CIPs |
| `name` | A sequence of printable, non-whitespace ASCII characters, interpreted by the resolver | `blackrock.com`, `blackrock.canton`, `partyHint::1220abcd...` |

Examples:

- `mainnet:dns:blackrock.com`
- `mainnet:dns:tmmf.lloyds.com`
- `mainnet:cns:blackrock.canton`
- `mainnet:lei:784F5XWPLTWKTBV3E584`
- `mainnet:party-id:partyHint::1220abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000abcd0000`

The `<name>` component is opaque to this standard and interpreted only by the resolver. This lets any external naming syntax — DNS dots, LEI codes, Party IDs with their `::` separators — embed without escaping.

#### 2.2 Network Discrimination

The `network` component is mandatory. TestNet names MUST NOT be confusable with MainNet names. Applications supporting multiple networks SHOULD display the network.

#### 2.3 The `party-id` Resolver (Built-in)

The `party-id` resolver is defined by this CIP and available on every network. Its `<name>` is the Canton Party ID verbatim. It guarantees that every Canton party always has at least one FQPN, before any name registration, with no external state and no trust beyond the Party ID itself.

### 3. Resolution

#### 3.1 Resolver Operations

Every resolver provides two logical operations. How they are transported and exposed is implementation-defined.

- resolve: name → the Party ID the name maps to, plus associated metadata claims and a validity horizon.
- reverseResolve: Party ID → the names this resolver knows for it.

#### 3.2 Forward Resolution

An application resolving a user-typed name matches it against each resolver in its configured, ordered list and returns the first successful result. Per-resolver ignore rules (patterns on the `<name>` component) MAY skip a resolver for matching names, addressing overlaps between resolvers when the application wants a specific delegation.

#### 3.3 Preferred-Name Resolution (Reverse Direction)

Given a Party ID, an application computes the preferred FQPN to display:

1. Retrieve the party's preferred-name credential from the Credentials Registry: issuer = holder = the party, claim `cip-cns/preferred-name` (claim key per the CNS 2.0 CIP).
2. Interpret the claim value as an FQPN and resolve it forward per Section 3.2, verifying it resolves back to the same Party ID. This round-trip verification prevents a party from claiming a name it does not hold.
3. If verified, display that name.
4. Otherwise, perform reverse resolution across the configured resolvers and select one result by the application's preference order — by default, lexicographic sort by (application priority override, user priority, application priority default).

### 4. Rendering

Rendered names follow a shared convention: a source icon followed by the human-readable name.

```
<icon> <display-name>
```

- For `dns`, `lei`, `ens`, and `party-id`, there is one standardized icon per resolver.
- For `cns`, the icon is the logo of the TLD under which the name is registered. Registrars self-publish their TLD logo, registered via a DSO-issued credential (`cip-cns/tld-logo-url`, per the CNS 2.0 CIP); applications retrieve the logo URL from the Credentials Registry at render time.
- The display name is the resolver's canonical form of `<name>` (e.g. `blackrock.com`, `blackrock.canton`, or a truncated Party ID prefix).
- Fallback chain when no name resolves: CNS 1.0 entry → Party ID prefix → truncated Party ID.

Assets are not named by FQPNs. An instrument identified by the token standard's `InstrumentId {admin: Party, id: Text}` renders as `<symbol> by <admin-rendered-name>` (e.g. `CC by <cns icon> dso.cns`), where the symbol is published by the admin through its token-standard metadata endpoint (`/registry/metadata/v1/instruments/{id}`), discovered via the admin's credentials (see the forthcoming CIP-56 amendment).

### 5. Integrating Additional Name Sources

New name sources enter the Canton Network either by defining a new resolver or by extending an existing one, in each case via a dedicated CIP. Such a CIP MUST specify:

- The resolver identifier as it appears in FQPNs (or the existing resolver being extended).
- The `<name>` syntax the resolver accepts.
- The naming authority and the governance under which its registrars operate.
- The procedure by which name-to-Party-ID bindings are established and, where the underlying authority can change externally, the re-verification cadence.
- The on-ledger credential encoding of name registrations, using the CN Credentials Standard. Names MUST be materialized as on-ledger credentials so that Daml code can reference them (import model, per the July 9, 2026 WG direction); reference-only integration without on-ledger state is acceptable only where no Daml logic depends on the name.
- The rendered-name convention, if it differs from `<name>` verbatim.

Registration credentials SHOULD carry at minimum the canonical FQPN, the network, and a validity horizon as claims. The claim-key prefix used by this document family (`cprp/`) is a working prefix, to be renamed `cip-<nr>/` on number assignment, consistent with `cip-cns/*`.

The first such source CIP is the Canton Imported Names CIP (Decentralized DNS Name Importer), which specifies DNS import in full and serves as the blueprint for LEI/vLEI, ENS, email, and further importers.

### 6. Backward Compatibility

Additive. CNS 1.0 names are preserved and resolve under the `cns` resolver in their existing display form. Party IDs continue to work as they always have; the `party-id` resolver exposes them as FQPNs. Applications that do not adopt this CIP are unaffected.

## Rationale

### Why exactly these four things

Representation, resolution, rendering, and integration are what require network-wide agreement: two applications must parse the same FQPN, obtain the same binding, show the user a consistent picture, and let new sources plug in predictably. Everything else — transports, endpoints, deployment, economics, composition policies — can differ between implementations without harming interoperability, so this CIP leaves it free.

### Why free-form `<name>`

Every naming system has its own syntax — DNS uses dots, LEI uses fixed-length codes, Party IDs use `::`. Normalizing them into a shared syntax either loses information or requires escaping that destroys readability. Ceding `<name>` syntax to each resolver puts the responsibility where the domain knowledge lives.

### Why three components and no registrar

The June 18–25 Working Group discussions judged a fourth `registrar` component to solve no problem that could not be solved by governance credentials: registrar identity, lifecycle, and approval are per-resolver governance concerns, and embedding them in the addressing format would complicate the format for no discovery benefit.

### Why first-match resolution and no composition

Name-selection UIs in the wild are always context-scoped (Leonid Rozenberg, June 25, 2026); a single ordered resolver list with ignore rules covers them. Cross-source result composition was part of earlier drafts of this proposal and has been removed: it standardized behavior that applications can layer on privately if they need it, at the cost of a materially larger standard.

### Why round-trip verification for preferred names

A preferred-name claim is self-issued. Verifying that the claimed FQPN actually resolves back to the claiming party costs one forward resolution and eliminates the entire class of display-name spoofing via false preference claims.

## Companion Documents

- CNS 2.0 CIP (Digital Asset) — the `cns` resolver, TLD registry, registrar governance, and registry economics.
- CN Credentials Standard CIP (Digital Asset, PR #204) — the credential registry that name registrations publish into.
- Party Profile Credentials CIP (PixelPlex, PR #169) — profile claim keys used in rendering.
- CIP-56 amendment (forthcoming, Digital Asset) — off-ledger API endpoint discovery via CN Credentials.
- Canton Imported Names CIP (Freename) — the DNS importer and blueprint for external name sources.
- `.canton` CIP (Axymos, PR #209) — allocation and governance of the `.canton` name space.
