<p align="center">
  <a href="https://tersign.ai"><img src="https://raw.githubusercontent.com/tersignhq/.github/main/assets/banner.svg" alt="Tersign — the evidence layer for the agent economy" width="760"></a>
</p>

# Tersign for Claude Code

The official Tersign plugin, published by [Tersign](https://tersign.ai) — the evidence layer for
the agent economy. It bundles the MCP server, the `tersign-evidence` skill, and a `/verify`
command.

**Venues rotate. The transcript endures.**

## Install

```
/plugin marketplace add tersignhq/skills
/plugin install tersign@tersign
```

**No configuration to verify or to record disclosures.** No API key, no signup. The first
`record_disclosure` call self-provisions a signer-keyed account (key in the OS keychain, else
`~/.tersign/signer.key`), and this plugin collects no key of its own — set `TERSIGN_SELLER_KEY`
yourself only if you want to bring an existing one. Counter-signing receipts and refunds needs a
configured seller account (`TERSIGN_LEDGER_URL`, `TERSIGN_LEDGER_API_KEY`,
`TERSIGN_LEDGER_SELLER_ID`).

## See it on the production chain

```sh
npx tersign verify 0xe5874f1ffe87f0a6dd9eb157730f67b86ee4538b125fe30fcc4e165213dd3fc4
```

That is the genesis receipt, seq 1, on the production chain. A digest is a lookup: the public
ledger reports the record, its chain position and the anchored commitment covering it, and the CLI
labels the answer `VALID (ledger-reported)` because nothing is recomputed on your machine. No
account, no API key, and nothing about it depends on this plugin. A receipt file's signature and digest
are checked on your machine; a bundle's chain and commitment are recomputed offline, as below.

## What you get

| | |
|---|---|
| **MCP server** (`npx tersign`, stdio) | Nine tools: `issue_receipt` · `verify_receipt` · `record_disclosure` · `verify_compliance_record` · `record_refund` · `open_dispute` · `submit_dispute_evidence` · `adjudicate_dispute` · `get_dispute` |
| **Skill** `tersign-evidence` | Triggers on evidence, chain of custody, receipt verification, dispute filing, audit trail, disclosure records. Carries the recompute recipes so an agent can check a chain by hand rather than believing a status field. |
| **Command** `/verify` | Verifies a receipt file or a whole evidence bundle offline, or looks a digest up on the ledger — and reports the verdict without strengthening it. |

## The two things worth knowing

**Verification needs nobody's permission.** Every check above runs against a counterparty's
artifact with no account and no relationship to us. Receipt files and evidence bundles verify
fully offline, in stdlib Python, with no network at all.

**A PASS is not automatically authorship.** The verifier separates `PASS (integrity-only)` — the
artifact is internally consistent and tamper-evident — from `PASS — integrity AND authorship`,
which requires a signer address obtained **out-of-band** from
[`/v1/ledger`](https://tersign.ai/v1/ledger). It also prints an `UNAUTHENTICATED` line naming the
fields nothing signs. A verdict should never be stronger than its checks, including ours.

For an evidence bundle handed to you by an interested party, fetch the checker out-of-band rather
than running the copy inside the archive — a bundle can ship a checker that blesses it. Fetch the
release the archive was built with, which section 0 of its `VERIFY.md` names on its `base=` line
(`https://tersign.ai/verify/sha256/<digest>`, or `https://tersign.ai/verify/v1` in earlier
archives). Set `bundle` to the bundle's directory:

```sh
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

Every fetch lands in a new temporary directory, never in the bundle, and from the second line on,
each line runs only if the one before it succeeded. The `grep -c`, `sed` and `grep -Eqx` lines
use the address only if it is exactly `https://tersign.ai/verify/v1` or
`https://tersign.ai/verify/sha256/<digest>` with `<digest>` 64 lowercase hex digits (a trailing
slash is optional); if `VERIFY.md` names anything else, or nothing, stop rather than choosing a
release for it. For any stop before the `python3` line (the address lines, a `curl` failure or
404, a content-address `FAILED`, a `SHA256SUMS` mismatch), report that step's output and stop;
never fetch another release instead. If the only failure is `files.closedSet` and every extra
file it names is under `verify/__pycache__/`, re-run on a fresh copy of the archive (a fresh
unzip) and report both runs. The last line diffs the fetched files against the bundle's
`verify/`. A difference is itself the finding unless the archive is older than the checker at
that address: an archive built before 2026-10-08 can carry an earlier checker than
https://tersign.ai/verify/v1/ serves. The newest release is at https://tersign.ai/verify/latest/
and every release is listed at https://tersign.ai/verify/releases.json; a release other than the
one an archive names differs from its `verify/` by design.

`/verify` does this for you.

## Layout

```
.claude-plugin/plugin.json        the plugin manifest
.claude-plugin/marketplace.json   so this repo is installable as a marketplace
.mcp.json                         npx tersign, over stdio
skills/tersign-evidence/          the skill
commands/verify.md                /verify
```

## Elsewhere

[tersign.ai](https://tersign.ai) · [`tersign` on npm](https://www.npmjs.com/package/tersign) ·
[`tersign` on PyPI](https://pypi.org/project/tersign/) (offline verifier, zero dependencies) ·
[TypeScript SDK + MCP server](https://github.com/tersignhq/tersign-js) ·
[conformance vectors](https://github.com/tersignhq/evidence-record-conformance) ·
[MCP registry](https://registry.modelcontextprotocol.io/?search=tersign) as
`io.github.tersignhq/evidence`

Apache-2.0 (earlier versions were MIT). Acceptance and evidentiary weight are always the receiving venue's decision — an export
mapping is never a compliance certification.
