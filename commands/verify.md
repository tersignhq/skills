---
description: Verify a Tersign receipt file or evidence bundle offline, or look a digest up on the ledger — always saying plainly what the result does and does not prove.
---

## Your task

Verify whatever the user named in `$ARGUMENTS`. Pick the path by what they gave you, run it, and
report the result **without strengthening it**.

### 1. Choose the path

| What they gave you | Command |
|---|---|
| A `0x…` 64-hex digest | `npx -y tersign@0.6.3 verify <digest>` — a lookup: asks the ledger whether it holds the record and whether its counter-signed chain holds, and prints `VALID (ledger-reported)` because nothing is recomputed on this machine. Report it with that label. Add `--ledger <url>` only if they name a different ledger; the default is printed either way. |
| A path to a receipt JSON file | `npx -y tersign@0.6.3 verify <path> --signer 0x<issuer>` — fully offline, touches no chain and no network. Take the issuer's address from out-of-band; without `--signer` the signer prints `UNAUTHENTICATED`, and your report keeps that word. |
| A standalone action-record file (`record` + `attestation`) | Not a CLI input: `npx -y tersign@0.6.3 verify` checks receipts. Verify it inside its evidence bundle (step 2), or offline with `verifyActionRecord(signed, issuerAddress)` from the `tersign` package (the discloser's signature only — no chain or counter-signature). |
| A directory holding `manifest.json` + `chain.json` (an evidence bundle) | Go to step 2. |
| Nothing | Run the genesis example so they see a live ledger answer: `npx -y tersign@0.6.3 verify 0xe5874f1ffe87f0a6dd9eb157730f67b86ee4538b125fe30fcc4e165213dd3fc4` |

### 2. Evidence bundles — fetch the checker out-of-band

A bundle ships a copy of `verify/` **inside itself**. That is fine for a worked example and wrong
for evidence handed over by an interested party: a bundle can ship a checker that blesses it. So
fetch the published release the archive was built with and run *that* copy. Section 0 of
`<bundle-dir>/VERIFY.md` names it on its one `base=` line: a content address
`https://tersign.ai/verify/sha256/<digest>` (every release since 2026-10-08), or
`https://tersign.ai/verify/v1` (earlier archives).

```bash
bundle=<bundle-dir>
[ "$(grep -c '^base=' "$bundle/VERIFY.md")" = 1 ] &&
base=$(sed -n 's/^base=//p' "$bundle/VERIFY.md") &&
printf '%s\n' "$base" | grep -Eqx 'https:/{2}tersign\.ai/verify/(v1|sha256/[0-9a-f]{64})/?' &&
base=${base%/} && OOB=$(mktemp -d) &&
curl -fsSL "$base/SHA256SUMS" -o "$OOB/SHA256SUMS" &&
case $base in */sha256/*) ( cd "$OOB" && echo "${base##*/}  SHA256SUMS" | shasum -a 256 -c ) ;; esac &&
for f in verify_bundle.py keccak.py secp256k1.py; do curl -fsSL "$base/$f" -o "$OOB/$f"; done &&
( cd "$OOB" && shasum -a 256 -c SHA256SUMS ) &&
python3 "$OOB/verify_bundle.py" "$bundle" &&
diff -r -x SHA256SUMS "$OOB" "$bundle/verify"
```

From the second line on, each line runs only if the one before it succeeded. For any stop before
the `python3` line (the address lines, a `curl` failure or 404, a content-address `FAILED`, a
`SHA256SUMS` mismatch), report that step's output and stop; never fetch another release instead.

- **The address.** The `grep -c`, `sed` and `grep -Eqx` lines accept exactly one `base=` line,
  and only if it is exactly `https://tersign.ai/verify/v1` or
  `https://tersign.ai/verify/sha256/<digest>` with `<digest>` 64 lowercase hex digits; a trailing
  slash is optional (`/{2}` is the `//` after `https:`). If they stop the chain,
  run `grep -n '^base=' <bundle-dir>/VERIFY.md`, report what it printed (nothing means the archive
  names no published checker), and stop. Do not pick a release for it: a different release can
  fail an honest archive.
- **The release.** For a content address, the `case` line checks that the fetched `SHA256SUMS`
  hashes to the digest in the address, before `shasum -c` checks the files against it.
- **The diff.** The last line prints nothing when the bundle's `verify/` is byte-identical to the
  release its VERIFY.md names. A difference is itself the finding unless the archive is older than
  the checker at that address: an archive built before 2026-10-08 can carry an earlier checker
  than `https://tersign.ai/verify/v1/` serves. The files at a content address never change, so
  that exception applies only when `base` is `https://tersign.ai/verify/v1`. Then read `createdAt`
  at the top level of `<bundle-dir>/manifest.json` (the verifier prints the same value on its
  `UNAUTHENTICATED` line). If it is earlier than 2026-10-08, report the published checker's
  verdict and report the difference as expected for an archive of that age, adding that
  `createdAt` is the archive's own unsigned claim. In every other case **a difference is itself
  the finding**, and you should say so loudly.

Never diff against `https://tersign.ai/verify/latest/`, or against `https://tersign.ai/verify/v1/`
unless VERIFY.md names it: a different release differs by design. Every release is listed at
https://tersign.ai/verify/releases.json.

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
into "something looks off". If the only failure is `files.closedSet` and every extra file it
names is under `verify/__pycache__/` or `oob/` (an earlier archive's own steps can leave either there), re-run on a fresh copy of the archive (a fresh unzip) and
report both runs.

### 4. If they want more

- Anchor proof for a digest: `curl -s https://tersign.ai/v1/receipts/<digest>/verify`
- Venue-shaped envelope: `curl -s "https://tersign.ai/v1/receipts/<digest>/envelope?venue=generic"`
  (`internet-court`, `kleros`, `uma` also serve)
- Whether acceptance follows is always the receiving venue's decision, never ours — say so if
  they ask what a court will do with it.
