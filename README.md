# AWC Block Explorer

Single-file, zero-dependency block explorer for the AWC chain (TRON-compatible DPoS). Just open `explorer.html` in a browser — no build step, no backend, no npm.

**English** | [中文](README_zh.md)

![node](https://img.shields.io/badge/node-none-blue) ![deps](https://img.shields.io/badge/dependencies-0-green)

## Features

| View | Route | Description |
|---|---|---|
| Home | `#/` | Chain status (height / block time / average interval / 40-block tx sparkline / producer distribution) + latest blocks (with interval column) + recent txs (type chips), 4s incremental auto-refresh |
| Block list | `#/blocks/<height>` | 20 blocks per page with pagination |
| Block detail | `#/block/<height\|hash>` | Header (incl. witness signature) + block tx table + prev/next navigation + raw JSON |
| Transaction | `#/tx/<hash>` | Confirmations (final at ≥19), type, amount, expiration, result, raw JSON; falls back to block lookup when a block hash is entered |
| Address | `#/addr/<T...>` | Balance / activation / bandwidth / nonce + SR badge + tx statistics + freeze & resource delegation details + vote details + latest 100 txs with in/out direction |
| Witnesses | `#/witness` | Active count / epoch / total votes / registration fee + SR table (vote bars, live 30-block production count, produced/missed, job status) |
| Network | `#/network` | Node info + P2P peer table (height delta, in/outbound) + full chain parameter table (27 items) |
| Assets | `#/assets` | Issued tokens / local wallets (balance + holdings) / smart contracts (gas, VM flags) + last-50-block token & contract activity scan |

Search auto-detects: plain number → block height; 64-char hex → transaction (falls back to block); `T...` → address.

## Quick start

**Option 1: open directly**

```bash
# just double-click explorer.html, or
python3 -m http.server 8000   # then visit http://localhost:8000/explorer.html
```

**Option 2: GitHub Pages**

Repo Settings → Pages → Source: `main` branch root, then visit:

```
https://<your-username>.github.io/<repo>/explorer.html
```

## RPC configuration

- Defaults to `https://mc.wanys.xyz:8901` — an HTTPS gateway (nginx) in front of the node's `:8900`; REST is auto-derived as `:8801` (the `REST = RPC − 100` rule still holds). Switch nodes from the top-right control (persisted in localStorage)
- **Why HTTPS**: GitHub Pages is served over `https://`, and browsers block `http://` calls from such a page as mixed content
- **First visit**: the gateway uses a self-signed certificate — open `https://mc.wanys.xyz:8901` once in your browser and accept the warning (Advanced → Proceed), after which this page will fetch normally
- Works from `file://` (CORS `*`)
- Requires a modern browser (BigInt etc. — any Chrome/Edge/Firefox/Safari from the last two years)

## API dependencies

All data is read-only, from two endpoint groups:

| Type | Address | Used for |
|---|---|---|
| TronRPC | RPC port (e.g. `:8900`) | `/wallet/*` — blocks, txs, accounts, witnesses, chain params, tokens |
| REST | **RPC port − 100** (e.g. `8800`, or `8801` behind the HTTPS gateway) | `/api/*` — P2P peers, local wallets, contract list |

Compatibility:

- **Node ≥ b27**: REST has CORS enabled — otherwise the network/assets REST cards degrade gracefully (everything else still works)
- **Node ≥ b26**: transactions include flat top-level fields (`type`, `type_name`, `from`, `to`, `amount`…) for precise type labels; older nodes fall back to nested `raw_data` parsing
- Address comparison is done on the decoded base58 payload, so non-standard-checksum addresses of the same account match correctly

## Companion

- [`payment-test.html`](payment-test.html) — payment testing page: QR code receiving, incoming-payment listener (banner + sound + session total), expected-amount alerts, sign-and-send loop test
- [`API_DOCUMENTATION.md`](API_DOCUMENTATION.md) — full node API reference (every endpoint used by these pages)

## Disclaimer

This repository is a read-only on-chain data viewer. It does not custody private keys and does not initiate transactions (the send feature in the payment test page requires your own private key and runs entirely in your local browser).
