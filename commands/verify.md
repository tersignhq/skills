---
description: Verify a Tersign receipt file or evidence bundle offline, or look a digest up on the ledger — always saying plainly what the result does and does not prove.
---

## Your task

Verify whatever the user named in `$ARGUMENTS`. Pick the path by what they gave you, run it, and
report the result **without strengthening it**.

### 1. Choose the path

| What they gave you | Command |
|---|---|
| A `0x…` 64-hex digest | `npx tersign verify <digest>` — a lookup: asks the ledger whether it holds the record and whether its counter-signed chain holds, and prints `VALID (ledger-reported)` because nothing is recomputed on this machine. Report it with that label. Add `--ledger <url>` only if they name a different ledger; the default is printed either way. |
| A path to a receipt JSON file | `npx tersign verify <path> --signer 0x<issuer>` — fully offline, touches no chain and no network. Take the issuer's address from out-of-band; without `--signer` the signer prints `UNAUTHENTICATED`, and your report keeps that word. |
| A standalone action-record file (`record` + `attestation`) | Not a CLI input: `npx tersign verify` checks receipts. Verify it inside its evidence bundle (step 2), or offline with `verifyActionRecord(signed, issuerAddress)` from the `tersign` package. |
| A directory holding `manifest.json` + `chain.json` (an evidence bundle) | Go to step 2. |
| Nothing | Run the genesis example so they see a live ledger answer: `npx tersign verify 0xe5874f1ffe87f0a6dd9eb157730f67b86ee4538b125fe30fcc4e165213dd3fc4` |

### 2. Evidence bundles — fetch the checker out-of-band

A bundle ships a copy of `verify/` **inside itself**. That is fine for a worked example and wrong
for evidence handed over by an interested party: a bundle can ship a checker that blesses it. So
fetch the verifier from the published location and run *that* copy:

```bash
base=https://tersign.ai/verify/v1
mkdir -p /tmp/tersign-oob && cd /tmp/tersign-oob
curl -fsSL "$base/SHA256SUMS" -o SHA256SUMS
for f in verify_bundle.py keccak.py secp256k1.py; do curl -fsSL "$base/$f" -o "$f"; done
shasum -a 256 -c SHA256SUMS          # published digests must match what you fetched
python3 /tmp/tersign-oob/verify_bundle.py <bundle-dir>
```

Then `diff -r /tmp/tersign-oob <bundle-dir>/verify` (ignoring `SHA256SUMS`). On an honest bundle
they are identical — **a difference is itself the finding**, and you should say so loudly.

Stdlib Python only; no install, no account.

### 3. Report it honestly — this is the part that matters

The verifier distinguishes two verdicts and you must preserve the distinction:

- **`PASS (integrity-only)`** — the bundle is internally consistent and tamper-evident, but the
  signer identity was read from the bundle itself. A forger can ship a self-consistent bundle
  under their own keys. **Do not call this "verified" without qualification.**
- **`PASS — integrity AND authorship`** — only when `--signer <address>` was supplied from
  **out-of-band**. The production ledger address is published at `https://tersign.ai/v1/ledger`;
  fetch it from there, never from the artifact under test.

The verifier also prints an `UNAUTHENTICATED` line naming the fields nothing signs (party id,
mode, createdAt). Carry those into your summary rather than dropping them — they are exactly the
fields a forger edits, and a verdict must never be stronger than its checks.

On a FAIL, name the failing check verbatim (`chain.acc`, `files.closedSet`,
`anchors.methodsDeclared`, …) and what that specific check means. Do not paraphrase a failure
into "something looks off".

### 4. If they want more

- Anchor proof for a digest: `curl -s https://tersign.ai/v1/receipts/<digest>/verify`
- Venue-shaped envelope: `curl -s "https://tersign.ai/v1/receipts/<digest>/envelope?venue=generic"`
  (`internet-court`, `kleros`, `uma` also serve)
- Whether acceptance follows is always the receiving venue's decision, never ours — say so if
  they ask what a court will do with it.
