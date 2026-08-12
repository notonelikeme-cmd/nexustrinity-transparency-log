# Nexus Trinity — Transparency Log

Cryptographic proof for every audit run Nexus Trinity's pipeline records, without
disclosing the actual finding content. This repo contains **only**:

- `log/entries.jsonl` — one line per audit run: `{entry_id, sha256_hash, ts}`
- `signatures/<entry_id>.sig` — detached GPG signature over the original entry's
  canonical JSON, signed by the key below
- `proofs/<entry_id>.ots` — [OpenTimestamps](https://opentimestamps.org) proof,
  anchoring the entry's hash into the Bitcoin blockchain — independent of Nexus
  Trinity, GitHub, or anyone else's say-so
- `proofs/<entry_id>.tsr` — a classical RFC 3161 timestamp from an independent
  timestamping authority (FreeTSA) — a second, deliberately different trust model
  (centralized rather than decentralized)
- `batches/<batch_id>.batch.bundle.json` — a [Sigstore](https://sigstore.dev) keyless
  signature (via `cosign`) over the combined digest of a whole batch of entries at
  once, bound to a real-time OIDC login and logged permanently in
  [Rekor](https://rekor.sigstore.dev), Sigstore's own public transparency log — a
  fourth witness, independent of Nexus Trinity, GitHub, GitLab, Bitcoin, and FreeTSA
  all at once. Batched rather than per-entry because Sigstore's keyless model requires
  a fresh interactive login for every signature by design (no long-lived key to
  steal) — proves the exact ordered set of entries in `entry_hashes` existed together
  at the time of signing, not a per-entry Merkle inclusion proof.

This repo is also mirrored to [GitLab](https://gitlab.com/nexus-trinity-io-group1/nexustrinity-transparency-log)
— a second custodian on genuinely different infrastructure, so a GitHub-specific
outage, ToS action, or account compromise doesn't take out the only public copy.

**What this deliberately does not contain:** target names, contract addresses,
findings, source code, or reports. That's proprietary and stays in the private
pipeline repo. This repo only proves *that* a record existed, unaltered, at a given
time, and *who* attested to it — not *what* it says.

## Identity

Every entry is signed by:

```text
Michael S Ross (Nexus Trinity) <security@nexustrinity.io>
Fingerprint: 899D 69D1 D63C E41C 5314  82DE 80A3 8CA6 4C32 F086
```

Public key: [`pgp-key.asc`](./pgp-key.asc) in this repo, and published at
[nexustrinity.io/.well-known/pgp-key.txt](https://nexustrinity.io/.well-known/pgp-key.txt)
— two independent locations, so you don't have to trust either one alone.

## How to verify an entry

The full entry content (target, finding, report) is not published here — that stays
private, disclosed only to the relevant program/client. What's public is a commitment:
the hash, signature, and timestamp proof for every entry, published *before* any
content is disclosed. If you're given a specific audit entry's content later (directly
from Michael, or as part of a shared report), you can independently confirm it matches
what was committed here at the time, rather than taking the disclosure's word for it:

```bash
# Import the key
gpg --import pgp-key.asc

# Reconstruct the canonical JSON from the disclosed entry (sorted keys, no whitespace,
# signature/ots_proof_path fields excluded — see agents/attestation.py's canonical_bytes())
# then verify:
gpg --verify signatures/<entry_id>.sig <reconstructed_canonical.json>

# Confirm the hash matches what's in log/entries.jsonl:
sha256sum <reconstructed_canonical.json>

# Verify the OpenTimestamps proof (requires the ots CLI: pip install opentimestamps-client)
ots verify proofs/<entry_id>.ots

# Verify the RFC 3161 timestamp (needs real OpenSSL's `ts` subcommand — on macOS the
# system `openssl` is LibreSSL and fails here; use Homebrew's: brew install openssl@3)
openssl ts -verify -in proofs/<entry_id>.tsr -queryfile proofs/<entry_id>.tsq \
  -CAfile freetsa_ca.pem -untrusted freetsa_tsa.crt

# Verify a batch cosign attestation (requires the cosign CLI)
cosign verify-blob --bundle batches/<batch_id>.batch.bundle.json \
  --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" \
  batches/<batch_id>.batch.json
```

A passing `gpg --verify` proves Michael Ross attested to that exact content and it
hasn't been altered since. A passing `ots verify` proves the hash existed at or before
the timestamp shown — anchored in the Bitcoin blockchain, not in anything Nexus
Trinity controls. Together: the commitment is public and immutable from the moment of
the audit run, even though the content isn't disclosed until later.

## Why this exists

Built in response to the [Digital Integrity Institute's defensibility
rubric](https://digitalintegrityinstitute.org/#rubric) — specifically criteria 3
(independent authority/tamper-evidence) and 4 (identity integrity), the two floor
criteria that decide whether a record survives a real challenge. See
[nexustrinity.io/transparency](https://nexustrinity.io/transparency) for the full
self-score, including what's still not addressed.
