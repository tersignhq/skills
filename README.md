<p align="center">
  <a href="https://tersign.ai"><img src="https://raw.githubusercontent.com/tersignhq/.github/main/assets/banner.svg" alt="Tersign — the evidence layer for the agent economy" width="760"></a>
</p>

# Tersign for Claude Code

The Tersign plugin for Claude Code, published by [Tersign](https://tersign.ai) — the evidence
layer for the agent economy. It bundles the MCP server, the `tersign-evidence` skill, and a
`/verify` command.

**Venues rotate. The transcript endures.**

## Install

```
/plugin marketplace add tersignhq/skills
/plugin install tersign@tersign
```

**No configuration to verify or to record disclosures.** No API key, no signup. The first
`record_disclosure` call self-provisions a signer-keyed account with a key generated on your
machine (see [What this plugin runs, sends and stores](#what-this-plugin-runs-sends-and-stores)).

## Settings

Every setting is optional. Leave them all empty and the server behaves exactly as `tersign` 0.6.3
does when started with no configuration.

| Setting | What it is | Needed for |
|---|---|---|
| `seller_key` (sensitive) | A 0x-prefixed 32-byte hex private key to sign with, instead of the generated one | Bringing a key you already use |
| `ledger_url` | Base URL of the ledger, normally `https://tersign.ai` | Counter-signed receipts, refunds, the dispute tools |
| `ledger_api_key` (sensitive) | API key of a Tersign seller account | Counter-signed receipts, refunds, respondent evidence |
| `ledger_seller_id` | Seller ID of that account | Counter-signed receipts, refunds |

Set them inside Claude Code with `/plugin configure tersign@tersign`, the plugin's configuration
dialog, which masks the sensitive values. The `/config` panel also lists `ledger_url` and
`ledger_seller_id`. From a shell, `claude plugin configure tersign@tersign` shows which settings
are set. Claude Code keeps the sensitive values in your operating system's credential store and
the others in your user `settings.json`.

The plugin passes the settings to the server as `TERSIGN_SELLER_KEY`, `TERSIGN_LEDGER_URL`,
`TERSIGN_LEDGER_API_KEY` and `TERSIGN_LEDGER_SELLER_ID`, empty when unset. Variables of those
names exported in your shell do not reach the server; use the settings instead. When a tool's
error names one of those variables, set the matching setting.

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
| **MCP server** (`npx -y tersign@0.6.3`, stdio) | Nine tools: `issue_receipt` · `verify_receipt` · `record_disclosure` · `verify_compliance_record` · `record_refund` · `open_dispute` · `submit_dispute_evidence` · `adjudicate_dispute` · `get_dispute` |
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
file it names is under `verify/__pycache__/` or `oob/` (an earlier archive's own steps can leave either there), re-run on a fresh copy of the archive (a fresh
unzip) and report both runs. The last line diffs the fetched files against the bundle's
`verify/`. A difference is itself the finding unless the archive is older than the checker at
that address: an archive built before 2026-10-08 can carry an earlier checker than
https://tersign.ai/verify/v1/ serves. The newest release is at https://tersign.ai/verify/latest/
and every release is listed at https://tersign.ai/verify/releases.json; a release other than the
one an archive names differs from its `verify/` by design.

`/verify` does this for you.

## What this plugin runs, sends and stores

**What it runs.** Claude Code starts the MCP server with `npx -y tersign@0.6.3`, over stdio. npx
downloads the [`tersign`](https://www.npmjs.com/package/tersign) package, version 0.6.3, and its
dependencies (`@modelcontextprotocol/sdk`, `viem`, `zod` and theirs) from the npm registry your npm
configuration names (https://registry.npmjs.org by default), then runs it on your machine. Its
source is at https://github.com/tersignhq/tersign-js. The plugin has no hooks and runs nothing else
on its own.

**What the MCP tools send.** `{ledger}` is `ledger_url` when set. When it is empty,
`record_disclosure` uses https://tersign.ai, and the tools that need a ledger URL return a
configuration error without any network request.

| Tool | Network request | What it sends |
|---|---|---|
| `verify_receipt`, `verify_compliance_record` | None | Nothing: they run offline |
| `record_disclosure` | `POST {ledger}/v1/disclose` | A disclosure record signed with your key: the agent ID, the disclosure kind, the medium and resource URL when given, the time, and a digest of the disclosure text. The text itself never leaves your machine. No API key is sent; the first call creates a free account bound to your key's address |
| `issue_receipt` | `POST {ledger}/v1/receipts`, only when `ledger_url`, `ledger_api_key` and `ledger_seller_id` are all set; otherwise none | The API key as a bearer token, the seller ID, the signed receipt (network, resource URL, payer address, time, transaction hash) and its compliance record (supply description, tax scheme and currency, principal when given, and the issuer name, jurisdiction and tax ID, which are `unnamed seller` and `unknown` unless `TERSIGN_ISSUER_NAME`, `TERSIGN_ISSUER_JURISDICTION` or `TERSIGN_ISSUER_TAX_ID` is set in the environment Claude Code runs in) |
| `record_refund` | `POST {ledger}/v1/refunds`, with the same three settings | The API key as a bearer token, the original receipt digest, the amount and the reason |
| `open_dispute` | `POST {ledger}/v1/disputes`, when `ledger_url` is set | A dispute signed with your key: the receipt digest, the reason, the claimed amount, the statement when given, and the time |
| `submit_dispute_evidence` | `POST {ledger}/v1/disputes/{digest}/evidence`, when `ledger_url` is set | Evidence signed with your key: the dispute digest, your role, and each artifact's kind, digest, time and note (digests only; the content stays with you). As respondent, also the API key as a bearer token when one is set |
| `adjudicate_dispute` | `POST {ledger}/v1/disputes/{digest}/adjudicate`, when `ledger_url` is set | The dispute digest, in the address |
| `get_dispute` | `GET {ledger}/v1/disputes/{digest}`, when `ledger_url` is set | The dispute digest, in the address |

The server makes no other network requests. Your signing key is never sent: only signatures made
with it are.

**Records are permanent.** The ledger is append-only and hash-chained, and its chain commitments
are anchored publicly, so a record it accepts (a disclosure, a receipt, a dispute, evidence or a
verdict) cannot be edited or deleted afterwards, and anyone holding the record's digest can look it
up. Put no personal data into an agent ID, a statement, a note, a supply description or any other
field you send.

**What the command and the skill run.** `/verify` tells Claude to run:

- `npx tersign verify <0xdigest>`, which asks the ledger (https://tersign.ai, or the one named with
  `--ledger`) for `GET /v1/receipts/<digest>/verify`, and `npx tersign verify <receipt.json>
  --signer 0x…`, which checks a receipt file offline. npx fetches `tersign` from the npm registry
  when it is not already installed.
- For an evidence bundle, the `curl` lines above: they download `SHA256SUMS`, `verify_bundle.py`,
  `keccak.py` and `secp256k1.py` from the https://tersign.ai/verify/ release the bundle's
  `VERIFY.md` names into a new temporary directory, check them with `shasum`, run
  `python3 verify_bundle.py` on your machine against the bundle (standard library only, no
  network), and `diff` the release against the bundle's own copy.
- When you ask for more, `curl` to `https://tersign.ai/v1/receipts/<digest>/verify` or
  `https://tersign.ai/v1/receipts/<digest>/envelope`.

The `tersign-evidence` skill runs nothing by itself. It instructs Claude to recompute chains rather
than trust a status field, to use the same public endpoints (`/v1/receipts/<digest>/verify`,
`/v1/receipts/<digest>/envelope?venue=…`) and `npx tersign verify`, to fetch a bundle's checker
from the address in its `VERIFY.md`, and to use the MCP tools above.

**What it stores.** When no `seller_key` is set, the server reads its signing key from the macOS
keychain (service `tersign-signer`), else from `~/.tersign/signer.key`. If neither exists, it
generates a key on first start and stores it in the keychain, or, where no keychain is available,
in `~/.tersign/signer.key` (mode 0600, in a 0700 directory). That key is your evidence identity:
back it up. Claude Code stores the plugin settings as described under [Settings](#settings), and
npm keeps the downloaded package in its cache (`~/.npm` by default). Nothing else is written,
apart from the temporary directory `/verify` creates for a bundle's checker.

## Layout

```
.claude-plugin/plugin.json        the plugin manifest, with the optional settings
.claude-plugin/marketplace.json   so this repo is installable as a marketplace
.mcp.json                         npx -y tersign@0.6.3, over stdio, settings passed as its environment
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
