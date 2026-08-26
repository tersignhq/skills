# Tersign Agent Skills

Official agent skills published by [Tersign](https://tersign.ai) — the evidence layer for the agent economy.

| Skill | What it does |
|---|---|
| [`tersign-evidence/`](tersign-evidence/) | Produce, verify, and submit counter-signed transaction evidence: no-trust receipt verification, dispute-ready evidence envelopes for external venues (Internet Court · Kleros ERC-1497 · UMA), counter-signed receipt issuance via the `tersign` npm SDK, independently attested agent action and disclosure records. |

## Install

Claude Code: copy `tersign-evidence/` into `.claude/skills/` (project) or `~/.claude/skills/` (user). The skill needs no account for verification; issuance uses the [`tersign`](https://www.npmjs.com/package/tersign) SDK (`npm i tersign`).

## Prove it works — one command, no account

```sh
npx tersign verify 0xe5874f1ffe87f0a6dd9eb157730f67b86ee4538b125fe30fcc4e165213dd3fc4 --ledger https://tersign.ai
```

Local EIP-712 signature recovery plus a public chain check against the production ledger's genesis receipt.

## Machine surfaces

| Surface | Address |
|---|---|
| Verify API | `GET https://tersign.ai/v1/receipts/{digest}/verify` |
| Envelope API | `GET https://tersign.ai/v1/receipts/{digest}/envelope?venue={internet-court\|kleros\|uma\|generic}` |
| llms.txt | https://raw.githubusercontent.com/tersignhq/tersign-js/main/llms.txt |
| MCP registry | `io.github.tersignhq/evidence` — start: `npx tersign` (stdio) |

MIT. SDK source: [tersignhq/tersign-js](https://github.com/tersignhq/tersign-js). Conformance: [evidence-record-conformance](https://github.com/tersignhq/evidence-record-conformance).
