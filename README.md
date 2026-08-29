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

**No configuration.** No API key, no account, no signup. The first call self-provisions a
signer-keyed account (OS keychain, else `~/.tersign/signer.key`), and this plugin collects no
key of its own — set `TERSIGN_SELLER_KEY` yourself only if you want to bring an existing one.

## Prove it before you trust it

```sh
npx tersign verify 0xe5874f1ffe87f0a6dd9eb157730f67b86ee4538b125fe30fcc4e165213dd3fc4
```

That is the genesis receipt, seq 1, on the production chain. Local EIP-712 signature recovery
plus a public chain check — no account, no API key, and nothing about it depends on this plugin.

## What you get

| | |
|---|---|
| **MCP server** (`npx tersign`, stdio) | Nine tools: `issue_receipt` · `verify_receipt` · `record_disclosure` · `verify_compliance_record` · `record_refund` · `open_dispute` · `submit_dispute_evidence` · `adjudicate_dispute` · `get_dispute` |
| **Skill** `tersign-evidence` | Triggers on evidence, chain of custody, receipt verification, dispute filing, audit trail. Carries the recompute recipes so an agent can check a chain by hand rather than believing a status field. |
| **Command** `/verify` | Verifies a digest, a receipt file, or a whole evidence bundle — and reports the verdict without strengthening it. |

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
than running the copy inside the archive — a bundle can ship a checker that blesses it:

```sh
base=https://tersign.ai/verify/v1
curl -fsSL "$base/SHA256SUMS" -o SHA256SUMS
for f in verify_bundle.py keccak.py secp256k1.py; do curl -fsSL "$base/$f" -o "$f"; done
shasum -a 256 -c SHA256SUMS
```

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

MIT. Acceptance and evidentiary weight are always the receiving venue's decision — an export
mapping is never a compliance certification.
