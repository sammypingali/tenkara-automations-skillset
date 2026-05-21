---
name: tenkara-supabase-setup
description: Wire up read-only access to the Tenkara prod Supabase database for a developer who lacks owner/admin Supabase access (so OAuth MCP is unavailable). Uses a pre-created `mcp_readonly` Postgres role + `@modelcontextprotocol/server-postgres` via the session pooler. Use when the user wants to connect Claude to the Tenkara prod database, set up Supabase MCP, query prod, or asks anything like "hook up the database", "connect to Supabase", "set up the Tenkara database".
---

# Tenkara Supabase — Read-Only Setup (Prod)

Connects Claude Code to the **Tenkara prod** Supabase project using a dedicated
`mcp_readonly` Postgres role. Read-only is enforced by the role's SELECT-only
grants — no INSERT/UPDATE/DELETE permitted at the database level.

**Why not Supabase OAuth MCP**: `mcp.supabase.com` requires owner/admin access.
Developers with member/developer roles can't complete that OAuth flow, so we use
`@modelcontextprotocol/server-postgres` with a pre-configured connection string instead.

## What this sets up

- **Project**: Tenkara prod (`lciwjbtbadjpkooufsvx`)
- **MCP name**: `supabase-tenkara-readonly`
- **Access**: Read-only via `mcp_readonly` Postgres role (SELECT only)

### Connection details

| Field | Value |
|---|---|
| Postgres role | `mcp_readonly` |
| Pooler host | `aws-1-us-east-2.pooler.supabase.com:5432` |
| Database | `postgres` |

**Why session pooler**: The direct hostname (`db.lciwjbtbadjpkooufsvx.supabase.co`)
is IPv6-only. The session pooler is IPv4-compatible and the right choice for MCP
connections. Username format for the pooler is `role.project_ref`.

## Prerequisite (admin only)

The `mcp_readonly` role must already exist with SELECT grants on the public schema.
If unsure, check with the dev team (ping us on slack in #platform-dev) before proceeding.

## Steps

### 1. Confirm Claude Code is installed

Ask: "Is Claude Code already installed and working on your machine?"

If no — direct them to https://claude.com/claude-code and stop until installed.

### 2. Add the MCP server

Have the user run:

```bash
claude mcp add --scope user supabase-tenkara-readonly \
  npx -- -y @modelcontextprotocol/server-postgres \
  "postgresql://mcp_readonly.lciwjbtbadjpkooufsvx:iuDX6zVVhvzeqzaPPHfa@aws-1-us-east-2.pooler.supabase.com:5432/postgres"
```

- `--scope user` — available in all Claude Code sessions, not tied to a folder
- `@modelcontextprotocol/server-postgres` — standard postgres MCP, no OAuth needed
- Username is `mcp_readonly.lciwjbtbadjpkooufsvx` — the `role.project_ref` format Supabase's pooler requires

**Conflicting scope?** If Claude warns about `supabase-tenkara-readonly` being
defined in multiple scopes, remove the old local entry first:
```bash
claude mcp remove supabase-tenkara-readonly -s local
```
Then re-run the add command.

### 3. Restart Claude Code

The MCP only loads on session start. Have the user quit and reopen Claude Code,
then confirm `/mcp` shows `supabase-tenkara-readonly` as **connected**.

If it shows failed/disconnected — see Troubleshooting.

### 4. Confirm reads work

Run a quick sanity check:

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name LIMIT 10;
```

Should return a list of Tenkara tables. If it errors, stop and troubleshoot before
telling the user setup is done.

### 5. Done

Tell the user:

> You're connected to Tenkara prod in read-only mode. The `mcp_readonly` Postgres
> role has SELECT-only grants — writes are blocked at the database level.

## Troubleshooting

- **`getaddrinfo ENOTFOUND db.lciwjbtbadjpkooufsvx.supabase.co`** → Wrong host —
  that's the direct (IPv6-only) hostname. Remove and re-add using the exact command
  in Step 2, which uses the session pooler.

- **`SASL authentication failed` / `password authentication failed`** → Password
  is wrong. Contact Andrew (andrew@trytenkara.com) for the current `mcp_readonly` credentials.

- **Conflicting scopes warning** → Remove the local entry:
  ```bash
  claude mcp remove supabase-tenkara-readonly -s local
  ```
  Re-add at user scope (Step 2) and restart.

- **MCP shows "failed" after restart** → Confirm `npx` is on PATH (`which npx`).
  If not, install Node.js. Also check the connection string is properly quoted in
  `~/.claude.json`.

## What NOT to do

- Don't use the Supabase HTTP MCP (`mcp.supabase.com`) for this user — they can't complete OAuth.
- Don't expose the connection string password in Slack, Notion, or shared docs.
- Don't widen the access scope — if the user needs writes, that's a conversation with Andrew.
