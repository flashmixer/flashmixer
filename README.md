# Flash Mixer — MCP Integration Guide

Connect any MCP-compatible AI agent to **Flash Mixer** to create and manage Bitcoin
mixing orders programmatically.

- **Endpoint:** `https://flashmixer.io/mcp`
- **Transport:** HTTP Streamable
- **Protocol:** JSON-RPC 2.0
- **Auth:** `Authorization: Bearer <YOUR_MCP_TOKEN>`

> *For educational and privacy purposes only.*

---

## 1. Overview

The MCP server is a thin, **stateless** bridge that lets AI agents use Flash Mixer's
public capabilities through a single endpoint (`/mcp`). It speaks standard MCP, so it
works with PyCharm AI Assistant, Claude Desktop, and any client that supports MCP over
HTTP Streamable.

It exposes **5 tools**:

| Tool | Purpose |
|---|---|
| [`get_pool_configuration`](#tool-get_pool_configuration) | Read current pool limits, fee ranges, delays, BTC/USD rate |
| [`calculate_exchange_fees`](#tool-calculate_exchange_fees) | Estimate total-to-send + fee breakdown |
| [`create_exchange_order`](#tool-create_exchange_order) | Create a mixing order (1–2 delayed payouts) |
| [`get_order_status`](#tool-get_order_status) | Check status, confirmations, payouts |
| [`trigger_payment_check`](#tool-trigger_payment_check) | Force an immediate payment re-check |

> 💡 Best practice: call `get_pool_configuration` first. It returns the **authoritative**
> valid ranges (min/max amounts, fee ranges, delay options) and the current BTC/USD rate,
> so your agent never hardcodes values that can change.

---

## 2. Authentication

Requests are authenticated with a **Bearer token** in the `Authorization` header:

```
Authorization: Bearer <YOUR_MCP_TOKEN>
```

- The token is **static**, issued per integration, and validated on **every** request.
- Missing/invalid token → **401 Unauthorized**.
- The token is a **secret** — never commit it, never expose it client-side in public apps,
  and rotate it if it leaks.

### Getting a token

To get an MCP access token, first create a manual order through the Flash Mixer web interface and complete the payment.

After the payment receives **3 blockchain confirmations**, your MCP authentication key will be generated and shown **once** on the order status page. Save it immediately and place it in your client config as shown below.

> Note: You only need the **Bearer token** generated for your order. Keep it private and do not share it publicly.

---

## 3. Client configuration

The config format is the standard MCP `mcpServers` map.

### PyCharm AI Assistant

**GUI:** Settings → Tools → AI Assistant → Model Context Protocol → add server:
- **URL:** `https://flashmixer.io/mcp` *(the trailing `/mcp` is required)*
- **Headers:** `{ "Authorization": "Bearer <YOUR_MCP_TOKEN>" }`

**Project-level file** (e.g. `.junie/mcp/mcp.json` in your repo):

```json
{
  "mcpServers": {
    "flash-mixer": {
      "url": "https://flashmixer.io/mcp",
      "headers": { "Authorization": "Bearer <YOUR_MCP_TOKEN>" }
    }
  }
}
```

### Claude Desktop

Add to your Claude Desktop MCP config:

```json
{
  "mcpServers": {
    "flash-mixer": {
      "url": "https://flashmixer.io/mcp",
      "headers": { "Authorization": "Bearer <YOUR_MCP_TOKEN>" }
    }
  }
}
```

### Any MCP HTTP client

Point it at `https://flashmixer.io/mcp`, transport **HTTP Streamable**, with the
`Authorization: Bearer <YOUR_MCP_TOKEN>` header.

---

## 4. List the tools

```bash
curl -X POST https://flashmixer.io/mcp \
  -H "Authorization: Bearer <YOUR_MCP_TOKEN>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{ "jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {} }'
```

Returns descriptions and input schemas for all 5 tools.

---

## 5. Tools

All tool calls use `method: "tools/call"` with `params.name` and `params.arguments`.

### Tool: `get_pool_configuration`

Get current pool limits, fee ranges, delays, and the BTC/USD exchange rate. Use this to
learn what parameters are valid before creating an order.

- **Parameters:** none.
- **Returns:** current BTC/USD rate + timestamp; for each pool — min/max amount, fee
  percent range, fixed fee (USD), delay range and options; general settings (max payout
  addresses, min payout amount, order TTL, required confirmations).

```json
{
  "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "get_pool_configuration", "arguments": {} }
}
```

### Tool: `calculate_exchange_fees`

Estimate total fees and the amount to send before creating an order. Fetches the current
BTC/USD rate and computes locally.

| Parameter | Type | Description |
|---|---|---|
| `pool` | string | `"standard"` or `"premium"` |
| `net_amount_btc` | string | Amount you want to **receive**, e.g. `"0.5"` |
| `fee_percent` | number | Your chosen fee % (must be within the pool's range — see `get_pool_configuration`) |

- **Returns:** desired-receive amount; fee breakdown (percentage fee in BTC, fixed fee
  USD→BTC, total fee); **amount to send**; current rate + timestamp; a summary
  (Send X → Receive Y after Z fee).

```json
{
  "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": {
    "name": "calculate_exchange_fees",
    "arguments": { "pool": "standard", "net_amount_btc": "0.5", "fee_percent": 3.0 }
  }
}
```

### Tool: `create_exchange_order`

Create a new Bitcoin mixing order with delayed payouts.

| Parameter | Type | Description |
|---|---|---|
| `pool` | string | `"standard"` or `"premium"` |
| `fee_percent` | number | Your chosen fee % within the pool's range |
| `payouts` | array | **1–2** payout objects (see below) |

Each `payouts[]` object:

| Field | Type | Description |
|---|---|---|
| `payout_address` | string | Destination BTC address (`bc1…`, `1…`, or `3…`) |
| `net_amount_btc` | string | Amount this address should **receive**, e.g. `"0.5"` |
| `delay_hours` | integer | Delay before this payout (0–72; ≥ 2 for Premium) |
| `payout_order` | integer | `1` or `2` |

- **Returns:** order details including the **order UUID**, the unique **deposit
  (payment) address**, expiry time, and the payout schedule.

```json
{
  "jsonrpc": "2.0", "id": 4, "method": "tools/call",
  "params": {
    "name": "create_exchange_order",
    "arguments": {
      "pool": "standard",
      "fee_percent": 3.0,
      "payouts": [
        { "payout_address": "bc1qexample1...", "net_amount_btc": "0.3", "delay_hours": 2, "payout_order": 1 },
        { "payout_address": "bc1qexample2...", "net_amount_btc": "0.2", "delay_hours": 12, "payout_order": 2 }
      ]
    }
  }
}
```

> Save the returned **order UUID** — it's how you (or your agent) check status later.
> There is no account; the UUID is the handle on the order.

### Tool: `get_order_status`

Check an existing order: payment status, confirmations, and payout details.

| Parameter | Type | Description |
|---|---|---|
| `order_uuid` | string | The UUID returned by `create_exchange_order` |

- **Returns:** status (`new`/`confirming`/`confirmed`/`notified`/`processing`/`completed`/
  `expired`), confirmation count (X/3), and per-payout status.

```json
{
  "jsonrpc": "2.0", "id": 5, "method": "tools/call",
  "params": { "name": "get_order_status", "arguments": { "order_uuid": "<order-uuid>" } }
}
```

### Tool: `trigger_payment_check`

Manually trigger payment verification for an order — useful right after sending, instead
of waiting for the automatic check.

| Parameter | Type | Description |
|---|---|---|
| `order_uuid` | string | The UUID to re-check |

- **Returns:** whether a payment was detected, with status, confirmation count, and the
  transaction hash if found.

```json
{
  "jsonrpc": "2.0", "id": 6, "method": "tools/call",
  "params": { "name": "trigger_payment_check", "arguments": { "order_uuid": "<order-uuid>" } }
}
```

---

## 6. Typical agent flow

1. `get_pool_configuration` → learn valid ranges + current rate.
2. `calculate_exchange_fees` → confirm the total-to-send for the user's desired receive amount.
3. `create_exchange_order` → get the deposit address + order UUID.
4. *(user sends BTC)*
5. `get_order_status` (or `trigger_payment_check` right after payment) → track to completion.

---

## 7. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `401 Unauthorized` | Missing/invalid Bearer token, or `Authorization` header stripped by a proxy |
| Endpoint not found | URL missing the trailing `/mcp` |
| Tool rejects `fee_percent` / amount | Value outside the pool's valid range — re-read `get_pool_configuration` |
| No tools listed | Client not using HTTP Streamable transport, or wrong endpoint |

---

## 8. Notes & limits

- The endpoint is **stateless** — no sessions; each call stands alone.
- Programmatic access is **rate-limited**; design agents to back off rather than hammer.
- Keep your **token secret**; rotate on exposure.
- Public, no-key capabilities (config, create, status, fee calc) cover normal use;
  some maintenance operations are restricted — your integration won't need them.
  
## 🌐 Web
- [flashmixer.io](https://flashmixer.io)
## 🔗 Mirrors
- [flashmixer.to](https://flashmixer.to)
- [flashmixer.co](https://flashmixer.co)
## 🧅 Tor
- [http://flashmxup6d3qesb72j3ciurm6v2tgjjhdffndiv35o2ifk62wv7njid.onion](http://flashmxup6d3qesb72j3ciurm6v2tgjjhdffndiv35o2ifk62wv7njid.onion)  
  *(NoJS recommended)*
## ✈️ Telegram
- [Flash Mixer](https://t.me/flashmixer_bot)
## 📧 Support
- [flashmixer@proton.me](mailto:flashmixer@proton.me)

> *For educational and privacy purposes only.*
