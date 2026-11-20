# 🌈 ZeroName .eth integration roadmap

## 🌈 Status and scope

**Status:** planning document. This roadmap does not claim that ENS support is implemented.

This work adds manually submitted ENS `.eth` names to the existing ZeroName registry while preserving the original Namecoin-backed `.bit` service.

The two name systems have different trust and update models:

- `.bit` remains backed by the existing Namecoin updater and its current registry format/behavior. Audit it before changing it; do not replace or regress it.
- `.eth` is submitted manually through the ZeroName social layer. A server-side processor verifies the name against Ethereum mainnet and publishes a signed registry update.
- A `.eth` record resolves to the Ethereum `0x` site identity used by ZirohNeth Content v2. It does not resolve to an IPFS CID or content hash.

Implementation is coordinated with the companion [ZirohNeth roadmap](https://github.com/Floflis/ZirohNeth/pull/13).

## 🌈 Goals

1. Preserve existing `.bit` resolution and Namecoin synchronization.
2. Allow a user to submit a `.eth` name through a social request form.
3. Verify the ENS name and its current destination using a public Ethereum RPC and the ENS Universal Resolver.
4. Verify registration validity and expiration independently of address resolution. An `addr()` result alone is not proof that a name is active.
5. Keep ENS Data as a secondary fallback, without allowing stale cached data to bypass expiration checks.
6. Publish `.eth` records beside `.bit` records in the ZeroName registry, using the repository's established signing and publication flow.
7. Display the mapped address, expiration, verification time, and verification status to users.

## 🌈 Coordinated PR sequence

### PR 1. Audit and preserve the Namecoin `.bit` server

- Trace the existing Namecoin updater, registry writer, signing, publication, and client lookup paths.
- Record the current registry schema and compatibility requirements before modifying it.
- Verify checkpoint initialization, transaction lookup assumptions, name updates, renewals, deletions/expiration, and reorganization behavior.
- Preserve existing `.bit` semantics and add regression tests before sharing registry-writing code with the ENS processor.
- Do not silently replace the Namecoin updater with an ENS-specific implementation.

**Acceptance:** existing `.bit` records continue to resolve unchanged; updater behavior and known limitations are documented and tested.

### PR 2. Adapt the ZeroName client for `.eth`

- Recognize normalized `.eth` names without changing `.bit` parsing.
- Resolve an active published `.eth` entry to its Ethereum `0x` address.
- Preserve the distinction between a ZeroNet-style `.bit` destination and an Ethereum site identity.
- Reject malformed addresses, unknown states, expired records, and records whose validity cannot be established under the published policy.
- Keep cache behavior bounded by the record's expiration and verification metadata.

**Acceptance:** `.bit` regression tests pass; valid `.eth` records return the exact published `0x` address; expired or unverifiable records fail closed.

### PR 3. Add the social `.eth` request form

- Let a user submit an ENS name and, optionally, an expected destination address.
- Validate basic input shape and normalize the ENS name using a standards-compliant ENS normalization library.
- Make clear that submission is a request, not an immediate registration or proof of ownership.
- Prevent duplicate/conflicting requests and provide a visible processing state.
- Apply request rate limits and abuse controls.

**Acceptance:** submissions are safely stored/queued; invalid names and malformed addresses are rejected; no submitted value is published without server-side verification.

### PR 4. Implement the ENS request processor

- Use a configurable public Ethereum mainnet JSON-RPC endpoint.
- Resolve the normalized name through the ENS Universal Resolver, including supported CCIP Read behavior.
- Obtain authoritative registration/expiration information from the relevant ENS registrar/registry state. Do not infer active registration from `addr(name)`.
- Verify that the resolved destination is a valid Ethereum address and compare it with the optional expected address.
- Keep ENS Data as a secondary fallback only where its returned fields, provenance, and freshness are sufficient for the specific check.
- If current validity/expiration cannot be established, mark the request unverifiable and do not authorize publication or resolution.
- Record the verification source, timestamp, relevant block number, expiration timestamp, and result.
- Do not run a local Ethereum node or a continuous ENS watcher. Verification is on demand, triggered by a request or explicit refresh.

**Acceptance:** processor tests cover active, expired, unregistered, changed-address, unavailable-RPC, stale-fallback, malformed, and conflicting results.

### PR 5. Integrate registry update, signing, and publication

- Store `.eth` entries alongside `.bit` entries using a backward-compatible schema or an explicitly versioned migration.
- Define a single serialized/atomic registry-write path so the Namecoin updater and ENS processor cannot overwrite each other's changes.
- Keep untrusted request handling separate from access to the ZeroName signing key.
- Only publish records that passed the complete verification policy.
- Sign and publish through the established ZeroName site publication mechanism.
- Retain enough metadata to explain the last verified state and to reject expired entries at lookup time.
- Preserve auditability: request ID, normalized name, verification evidence metadata, and publication revision.

**Acceptance:** concurrent updates do not lose records; invalid requests cannot reach the signed registry; published content verifies; existing `.bit` records remain intact.

### PR 6. Security, expiration, conflicts, and compatibility tests

- Expiration boundary tests, including the exact expiry instant and any registrar grace-period semantics.
- ENS destination changes between submission and processing.
- ENS name transfer/resolver changes and re-registration.
- Expired names whose resolver still returns an old address.
- RPC outages, timeouts, provider disagreement, and stale ENS Data responses.
- Duplicate requests, conflicting destinations, replay, and concurrent registry updates.
- Invalid signatures, malformed registry records, cache invalidation, and `.bit` regression coverage.
- End-to-end verification from social submission to signed publication and client resolution.

**Acceptance:** no expired or unverifiable name is resolved; no stale fallback can extend validity; `.bit` behavior remains compatible.

## 🌈 Registry and validity contract

The exact field names and migration strategy must be finalized after auditing the existing ZeroName registry consumers. The following is a conceptual contract, not a committed schema:

- Name: ENS-normalized `.eth` name.
- Destination: Ethereum `0x` address, normalized and checksum-validated.
- Status: explicit active, expired, pending, rejected, or unverifiable state.
- Expiration: authoritative Unix timestamp, when available.
- Last verification: timestamp and source.
- Verified block: Ethereum block number/hash where applicable.
- Record revision: publication/update identifier.

**Hard rule:** expired names must not resolve. If the verifier cannot establish current validity, the record must not be treated as active. ENS Data is a fallback source, not permission to ignore stale or unknown expiration state.

A future expiration timestamp in an old cached response is not, by itself, proof that the name remains registered now. The implementation must account for registrar state, renewal, transfer, resolver changes, and re-registration.

## 🌈 Operational and security boundaries

- No local Ethereum node is required.
- No background ENS watcher is required; refresh is manual/on-demand.
- Public RPC endpoints must be configurable and failure-aware. Do not hard-code a single provider as an availability guarantee.
- ENS normalization must use a standards-compliant implementation, not lowercase-only conversion.
- The request processor must not hold the publication private key unless the repository's existing architecture makes separation impossible and the risk is explicitly reviewed.
- Never trust the submitted destination, client cache, or ENS Data cache as proof.
- Never publish an entry if address resolution and registration validity checks disagree or cannot be completed.
- The ENS `addr()` destination is the target identity. Registrar owner/controller and resolver destination are distinct concepts.

## 🌈 Central signer and trust model

The historical ZeroName pipeline is not fully decentralized at the publication-authority layer. The Namecoin blockchain is the source of `.bit` registration data, but the traditional updater runs on an operator-controlled machine: it connects to a local Namecoin RPC endpoint, reads the ZeroName site's registry, and invokes `siteSign ... --publish` with a configured site private key when the registry changes. The P2P network distributes the signed site, but possession of the site signing key authorizes registry updates.

Keep these roles distinct:

- **Namecoin / ENS:** authoritative blockchain state for the respective name system.
- **ZeroName registry:** a published index/cache of mappings, not the underlying name system.
- **Updater / ENS verifier:** reads and validates upstream state.
- **Publisher / signer:** authorizes and publishes a new registry revision.
- **ZeroName client plugin:** consumes the published registry; it does not need to operate the central updater or hold its key.

This is decentralized distribution with a centralized (or small trusted-group) write authority. It is not accurate to describe the complete ZeroName naming path as trustless.

For the `.eth` implementation:

- Treat social submissions and the request processor as untrusted inputs. They must not hold or access the publication key.
- Separate request intake, ENS verification, registry mutation, and signing/publication into explicit trust boundaries.
- Make publication auditable: retain request ID, normalized name, verification result, RPC/provider, chain ID, block number/hash where available, expiry, signer/publication revision, and timestamps.
- Protect the signing key, document backup/rotation/recovery, and ensure a compromised request processor cannot sign arbitrary mappings.
- Preserve the distinction between the authoritative ENS state and ZeroName's signed snapshot. A valid ZeroName signature proves which publisher authorized the snapshot; it does not independently prove that every ENS claim in it is true or current.
- Document the residual trust assumption in both server and client UX/security notes.

A future decentralization track may explore independent verifiers, public audit/transparency logs, multiple publishers, or threshold/multisignature authorization. These are not silently assumed to exist in this roadmap and must be designed and tested separately.

## 🌈 Out of scope

- IPFS, content-hash, or gateway resolution.
- A local Ethereum execution node.
- Continuous ENS monitoring or automatic renewal.
- Automatic publication based only on a social submission.
- Changes to the meaning or destination format of existing `.bit` records.

## 🌈 Delivery rule

The ZeroName and ZirohNeth changes are coordinated but independently reviewable. The registry contract and expiry policy must agree before either client-side or server-side implementation is considered complete. Each implementation PR must include tests and document any behavior that remains unsupported.
