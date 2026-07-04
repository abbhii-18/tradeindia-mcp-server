# TradeIndia MCP Server

> Connect Claude AI directly to your TradeIndia buyer inquiries — no OAuth, no manual exports, no copy-pasting leads.

Built by [abhijeetbuilts](https://abhijeetbuilts.tech) · Powered by n8n + Nginx + Anthropic MCP

---

## What This Does

This project exposes your TradeIndia buyer inquiries as a **live MCP tool** that Claude can call directly inside a conversation.

Instead of logging into TradeIndia, downloading a CSV, and manually reviewing leads — you just ask Claude:

> *"Get my TradeIndia leads from the last 2 days"*

And Claude fetches, parses, and presents them instantly — names, companies, mobile numbers, messages, product interest, city, state — everything clean and readable.

---

## Architecture
Claude (claude.ai)
│
│  MCP over HTTPS
▼
Nginx (IP Allowlist — Anthropic IPs only)
│
▼
n8n MCP Server Trigger
│
▼
n8n Subworkflow — Get Inquiries
│
├── HTTP Request → TradeIndia Inquiry API
├── Code Node → Parse + Clean leads
└── Code Node → Aggregate all leads into single JSON

**Key design decision:** No OAuth server required. The MCP endpoint is locked at the Nginx layer to Anthropic's published outbound IP range (`160.79.104.0/21`). Only Anthropic's servers can physically reach the URL — everyone else gets a `403`.

---

## Stack

| Component | Role |
|-----------|------|
| **n8n** (self-hosted, Docker) | MCP Server Trigger + workflow logic |
| **Nginx** | Reverse proxy + Anthropic IP allowlist |
| **TradeIndia Inquiry API** | Lead data source |
| **Claude (claude.ai)** | MCP client — calls the tool in conversation |
| **Hostinger VPS** (Ubuntu 24.04) | Hosting infrastructure |

---

## Project Structure
tradeindia-mcp/
├── README.md
├── workflows/
│   ├── 1-tradeindia-get-inquiries-subworkflow.json   # Fetches + cleans leads from TradeIndia API
│   └── 2-tradeindia-mcp-server.json                  # MCP Server Trigger + tool definition
└── nginx/
└── mcp-location-block.conf                       # Nginx config snippet to add to your server block

---

## Prerequisites

- Self-hosted n8n instance (Docker recommended) with a public domain
- Nginx as reverse proxy in front of n8n
- TradeIndia account with Inquiry API access (`userid`, `profile_id`, `key`)
- Claude Pro or Team account (custom connectors require Pro+)

---

## Setup Guide

### Step 1 — Get Your TradeIndia API Credentials

Log into TradeIndia → **Inquiries & Contacts → My Inquiry API**

You need three values:
- `userid`
- `profile_id`
- `key`

---

### Step 2 — Import the n8n Workflows

Import in this exact order (file 1 must exist before file 2 can reference it):

1. Go to your n8n instance → top-right menu → **Import workflow**
2. Import `workflows/1-tradeindia-get-inquiries-subworkflow.json`
3. Import `workflows/2-tradeindia-mcp-server.json`

After importing both:
- Open the **TradeIndia MCP Server** workflow
- Double-click the `get_tradeindia_inquiries` node
- In the **Workflow** field, reselect **"TradeIndia - Get Inquiries (Subworkflow)"** from the dropdown (n8n assigns new IDs on import, so this re-link is required)

---

### Step 3 — Add Your TradeIndia Credentials

Open **`TradeIndia - Get Inquiries (Subworkflow)`** → double-click the **HTTP Request** node → update the query parameters:
userid     → your TradeIndia userid
profile_id → your TradeIndia profile_id
key        → your TradeIndia API key

> **Security tip:** Move these into an n8n credential rather than leaving them as inline node values, especially if you share the workflow JSON.

---

### Step 4 — Configure the Date Range

The subworkflow's HTTP Request node uses these date expressions by default (last 24 hours):
from_date: {{ $now.setZone('Asia/Kolkata').minus({ hours: 24 }).toFormat('yyyy-MM-dd') }}
to_date:   {{ $now.setZone('Asia/Kolkata').toFormat('yyyy-MM-dd') }}

Adjust `minus({ hours: 24 })` to whatever window suits your use case (e.g. `{ days: 7 }` for last 7 days).

---

### Step 5 — Add the Nginx IP Allowlist

Add this block **above** your `server { }` blocks:

```nginx
geo $anthropic_allowed {
    default 0;
    160.79.104.0/21 1;
    2607:6bc0::/48  1;
}
```

Add this location block **inside** your n8n `server { listen 443 ssl; }` block:

```nginx
location /mcp/ti-9f2c7a41e8b04d1a-mcp {
    if ($anthropic_allowed = 0) {
        return 403;
    }
    proxy_pass http://localhost:5678;
    proxy_http_version 1.1;
    proxy_buffering off;
    gzip off;
    chunked_transfer_encoding off;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 3600s;
}
```

Then reload Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

### Step 6 — Activate the MCP Workflow

In n8n, open **TradeIndia MCP Server** → toggle **Active** to ON.

Copy the Production URL from the MCP Server Trigger node:
https://your-n8n-domain.com/mcp/ti-9f2c7a41e8b04d1a-mcp

---

### Step 7 — Connect to Claude

1. Go to **claude.ai → Settings → Connectors → Add custom connector**
2. Paste your Production MCP URL
3. Click **Add**

No OAuth flow appears — Nginx handles the gatekeeping at the IP level.

---

## How the IP Allowlist Replaces OAuth

Normally, connecting a custom MCP server to claude.ai requires implementing OAuth 2.0 — authorization URLs, token exchange endpoints, refresh token handling. That is significant infrastructure.

Instead, this project uses infrastructure-level access control:

- Anthropic publishes its stable outbound IP ranges
- Nginx checks every incoming request's source IP before it reaches n8n
- Non-Anthropic IPs receive `403 Forbidden` — n8n never sees the request
- The MCP endpoint requires no authentication layer because only Anthropic's servers can physically reach it

**One-liner:** *Lock the door at the network level, so you don't need a lock on the app level.*

---

## Sample Output
Found 4 TradeIndia inquiries:

Rajesh Kumar — Rajesh Packaging Pvt Ltd
Mobile: 9876543210 | City: Surat, Gujarat
Product: BOPP Tape | Message: "Need 5000 rolls monthly, please send best price"
Priya Exports — Mumbai
Mobile: 9812345678
Product: Stretch Film | Message: "Looking for bulk supplier"


---

## Limitations

- TradeIndia's Inquiry API is read-only — fetch only, no reply or status update
- IP allowlist works for claude.ai and Claude Desktop web connectors. Claude Code CLI connects from your local machine and will get a 403
- Anthropic's IP range is stable but not permanent — check [Anthropic's IP reference](https://platform.claude.com/docs/en/api/ip-addresses) for updates

---

## Author

**Abhijeet Singh** — AI Automation Consultant
[abhijeetbuilts.tech](https://abhijeetbuilts.tech)

Building AI systems that capture leads, follow up automatically, and remove manual work.

---

## License

MIT — use freely, attribution appreciated.
