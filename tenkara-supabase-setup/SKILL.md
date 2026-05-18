---
name: tenkara-supabase-setup
description: Wire up read-only access to the Tenkara prod Supabase database via the remote Supabase MCP. Use when the user wants to connect Claude (or SuperAgent) to the Tenkara prod Supabase database, set up Supabase MCP, query prod, or asks anything like "hook up the database", "connect to Supabase", "set up the Tenkara database". HARD REQUIREMENT — this setup is NOT complete until Step 5 has been executed and writes have been observed to fail with a server-side Postgres error — either `permission denied` (42501) or `read-only transaction` (25006). Do not skip Step 5. Do not declare success based on configuration alone.
---

================================================================================
                          !!  STOP — READ THIS FIRST  !!
================================================================================

  THIS SETUP IS NOT DONE UNTIL YOU HAVE PERSONALLY RUN A WRITE QUERY AGAINST
  THE DATABASE AND WATCHED IT GET REJECTED BY POSTGRES — with either
  `permission denied` (42501) or `read-only transaction` (25006).

  - Configuration looking right is NOT verification.
  - The URL containing `read_only=true` is NOT verification.
  - The MCP saying "connected" is NOT verification.
  - The user saying "looks good" is NOT verification.

  The ONLY thing that counts as verification is Step 5:
  an UPDATE and an INSERT, both returning a server-side permission error.

  If you skip Step 5, you have failed this task — full stop.

================================================================================

# Tenkara Supabase — Read-Only Setup (Prod)

This skill walks the user through connecting Claude Code to the **Tenkara prod** Supabase project in **read-only mode**. The user may be non-technical — be patient, explain each step, and confirm before moving on.

**Critical rule (repeated because it matters)**: You MUST NOT declare this setup complete until you have personally executed a write against the database and observed it fail with a server-side Postgres error — either `permission denied` (SQLSTATE `42501`) or `read-only transaction` (SQLSTATE `25006`). No exceptions. See Step 5.

## What this sets up

- **Project**: Tenkara prod (`lciwjbtbadjpkooufsvx`)
- **Access**: Read-only (server-enforced — must be verified, not assumed)
- **Auth**: Supabase OAuth in the browser (no passwords, no tokens to copy)

### Project reference

| Field | Value |
|---|---|
| project_ref | `lciwjbtbadjpkooufsvx` |
| Verification target material id | `9845dfd4-99fa-5971-85e4-94436838258a` (use directly, do not look up) |

## Steps

### 1. Confirm the user has Claude Code installed

Ask: "Is Claude Code already installed and working on your machine?"

If no — direct them to https://claude.com/claude-code and stop here until installed.

### 2. Add the MCP server

Run this exact command for the user (or have them paste it into their terminal):

```bash
claude mcp add --transport http --scope user supabase-tenkara-prod 'https://mcp.supabase.com/mcp?project_ref=lciwjbtbadjpkooufsvx&read_only=true'
```

What each piece means (explain if asked):
- `--transport http` — uses Supabase's hosted MCP server (no local install)
- `--scope user` — available in every Claude Code session, not tied to one folder
- `read_only=true` — must be present; this is the flag we are about to verify

### 3. Authenticate

In Claude Code, run:

```
/mcp
```

Find `supabase-tenkara-prod` and select **Authenticate**. A browser window opens — log into Supabase with the account that has access to the Tenkara prod project. Approve the connection.

Back in Claude Code, `/mcp` should now show the server as **connected**. Do not proceed until it does.

### 4. Verify reads work

Call the Supabase MCP `list_tables` tool (schemas=["public"], verbose=false).

**Expected**: A list of tables including `users`, `suppliers`, `materials`, `orders`, etc.

If this fails: stop and troubleshoot auth before going further. Reads not working = wrong account, wrong project_ref, or OAuth wasn't completed.

### 5. !! VERIFY WRITES ARE BLOCKED — MANDATORY, NON-NEGOTIABLE, DO NOT SKIP !!

> **This step is the entire point of the skill.** Skipping it = failing the task.
> Steps 1–4 prove the connection exists. Only Step 5 proves the connection is safe.
> Read-only is a claim until you have seen the server reject a write.

This is the only step that proves read-only is real. You MUST run it yourself via the MCP tools. You MUST NOT accept the user's word, the URL's appearance, the presence of `read_only=true` in the config, or the absence of obvious problems as proof.

**Procedure** — execute these in order using the Supabase MCP `execute_sql` tool:

1. **Target row.** Use this fixed prod material id directly. Do NOT search for one:
   ```
   id = '9845dfd4-99fa-5971-85e4-94436838258a'
   ```
   (Optionally confirm it exists with `SELECT id, name FROM materials WHERE id = '9845dfd4-99fa-5971-85e4-94436838258a';` — but the write attempt below works either way; if the row doesn't exist the UPDATE simply affects zero rows when *not* blocked.)

2. **Attempt a write** — append an `s` to the material's name:
   ```sql
   UPDATE materials
   SET name = name || 's'
   WHERE id = '9845dfd4-99fa-5971-85e4-94436838258a';
   ```
   This is the canonical verification write for this skill. It is a real mutation — if it succeeds, the database actually changed, which is exactly why we expect it to fail. A successful UPDATE here means you have just renamed a real prod material. Stop and revert immediately if that happens.

3. **Inspect the result.** Exactly one of two outcomes is acceptable:

   - ✅ **PASS**: The call returns a server-side Postgres error — **either**:
     - `permission denied` (SQLSTATE `42501`, e.g. `ERROR: 42501: permission denied for table materials`) — role lacks write grant, **or**
     - `read-only transaction` (SQLSTATE `25006`, e.g. `ERROR: 25006: cannot execute UPDATE in a read-only transaction`) — session is in `default_transaction_read_only=on` mode.

     Both are genuine server-side rejections coming back from Postgres (not a client-layer refusal). Either one means read-only is server-enforced and setup is verified.
   - ❌ **FAIL**: The call returns success, rows-affected, or any non-error response. **Read-only is NOT enforced.** Stop. Do not tell the user setup is done. Go to Troubleshooting → "Writes succeeded".

4. **Belt-and-suspenders second check** — try an INSERT into a different table:
   ```sql
   INSERT INTO authorized_operators (email) VALUES ('verify-readonly-test@example.invalid');
   ```
   Must also fail with a server-side Postgres error — either `permission denied` (`42501`) or `read-only transaction` (`25006`). If it succeeds, treat as FAIL above.

5. **Only after BOTH writes have been observed to fail** with permission errors may you tell the user: "Verified read-only — both UPDATE and INSERT were rejected by the server."

If the MCP refuses to run `UPDATE` / `INSERT` at the client layer (returns "this server is read-only" without contacting the database) that is NOT sufficient — it proves the client flag, not the server enforcement. Note this to the user and request a different verification path (e.g. running the SQL via Supabase SQL Editor logged in as the same role, if available). Do not declare success.

### 6. Confirm to the user (only after Step 5 passes)

Before you tell the user "done", you MUST be able to fill in this checklist truthfully. If any box is unchecked, you are not done — go back.

```
VERIFICATION CHECKLIST (all four must be true):
[ ] I called execute_sql with an UPDATE that appends 's' to material id
    '9845dfd4-99fa-5971-85e4-94436838258a'.
[ ] That UPDATE returned a server-side Postgres error — either
    "permission denied" (42501) or "read-only transaction" (25006).
[ ] I called execute_sql with an INSERT into `authorized_operators`.
[ ] That INSERT returned a server-side Postgres error — either
    "permission denied" (42501) or "read-only transaction" (25006).
```

Only when all four are checked, tell the user verbatim:

> Setup verified. I ran a live UPDATE and a live INSERT against the database. Both were rejected server-side by Postgres (either `permission denied` / `42501`, or `read-only transaction` / `25006` — both are valid server-enforced rejections). Read-only is enforced server-side — you're safe to use it.

If you cannot truthfully tick all four boxes, tell the user exactly what happened and that the setup is **not** verified.

## Troubleshooting

- **"command not found: claude"** → Claude Code isn't installed or not on PATH.
- **OAuth window doesn't open** → Make sure the user is signed into Supabase in their default browser first, then retry `/mcp` → Authenticate.
- **"connection failed" in /mcp** → Re-run the `claude mcp add` command; the URL may have been mistyped. The full URL must be wrapped in single quotes because of the `&`.
- **Writes succeeded in Step 5** → The URL is missing `read_only=true`, OR the authenticated Supabase role has elevated privileges that override it. Immediately:
  1. `claude mcp remove supabase-tenkara-prod`
  2. Re-add with the exact command in Step 2 (verify `read_only=true` is present and the URL is quoted).
  3. Re-authenticate via `/mcp`.
  4. Re-run Step 5 from scratch.
  5. If writes still succeed after a clean re-add, **stop and contact Andrew (andrew@trytenkara.com)** — do not use the connection.

## What NOT to do

- Don't give the user a Postgres connection string or service-role key — the MCP OAuth flow is the whole point. No secrets need to change hands.
- Don't suggest installing the Supabase CLI — not needed for read-only MCP usage.
- Don't try to widen the access scope. If the user needs writes, that's a separate conversation with Andrew (andrew@trytenkara.com).
- Don't skip Step 5. The presence of `read_only=true` in the URL is not proof — only an observed server-side Postgres rejection (`permission denied` / `42501`, or `read-only transaction` / `25006`) on a real write attempt is proof.

## For SuperAgent users

If the user is hooking this up to SuperAgent (https://github.com/SkillfulAgents/SuperAgent) instead of Claude Code, the URL is the same but goes into SuperAgent's MCP config block. The OAuth flow is identical. **Step 5 verification still applies** — run the same UPDATE / INSERT test through SuperAgent and confirm both are rejected before relying on the connection.
