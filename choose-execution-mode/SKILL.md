---
name: choose-execution-mode
description: Decide how a Tenkara automation should interact with an external system — via API/connector, Browser Use with Container host, Browser Use with BrowserBase host, or Computer Use. Invoke before building any new automation in SuperAgent, when an agent needs to pick how to reach a target system, or whenever you hear "should I use BrowserBase / Computer Use / API for X", "what's the right tool for scraping Y", "how do I get the agent to do Z". Wrong choice creates IP bans, brittle desktop automation, or unnecessary infrastructure cost.
metadata:
  author: Tenkara
  version: "0.1"
---

# Choose Execution Mode

Tenkara automations interact with three classes of system: things we own (Tenkara platform, internal dashboards), things we have accounts on (Missive, Gmail, Notion, Slack, BrowserBase, Tenkara internal), and the open web (marketplaces, supplier sites, LinkedIn, search engines). Each class has a correct execution mode. Picking wrong wastes weeks: too aggressive and you get IP bans and brittle scripts, too cautious and you ship slow agents that need infrastructure they don't need.

## Decision tree

Run through this in strict order. Stop at the first **yes**.

### 1. Does the target have an API or a SuperAgent connector?

Use the **API / connector**. No browser of any kind.

Examples that always go this route: Missive, Gmail, Notion, Slack, the Tenkara platform itself, BrowserBase (its own management API), Asana, HubSpot, Linear, Supabase, any standard SaaS with an officially supported OAuth or token-based API.

Why this beats browser flows: 10–50× faster, far more stable, machine-readable responses, no anti-bot defenses fighting you, no IP concerns. Always check for an API first.

### 2. Does the target have a web interface AND is it a system you own or have a legitimate logged-in session on?

Use **Browser Use** with **Container (built-in) host**.

Examples: the Tenkara admin/ops panel if no internal API exists for the action, an internal dashboard with no API, a SaaS tool whose API doesn't expose the action you need but its web app does.

Container is fine here because there's no anti-bot defense to fight and no IP-ban risk on systems you own or pay for.

### 3. Does the target have a web interface AND is it an external public site you don't own (especially at any volume)?

Use **Browser Use** with **BrowserBase host**.

Examples: marketplace scraping, supplier website crawls, Google/Bing searches, LinkedIn/Apollo/Hunter/RocketReach contact discovery, business-registry lookups, public-pricing scrapes, any "find me suppliers of X" task.

Why BrowserBase, not Container: requests originate from BrowserBase's IPs, not yours or SuperAgent's. Critical for anything you'd run more than ~10x/day. Tenkara's tech advisor Ben specifically recommended this path in May 2026 to avoid IP blacklisting.

### 4. No API. No web interface. Only a native desktop app?

Use **Computer Use** as a last resort.

This is the only mode that drives your real desktop — actual mouse, real screen, real apps. Slow, fragile, occupies your screen, can't run unattended on a schedule.

Before reaching for it, double-check the target really has no web/API path. Almost every modern tool does. As of May 2026, nothing in the Tenkara automation list (P0–P4, ONB, Dashboard, General) requires Computer Use.

## Anti-patterns — common wrong defaults

These are mistakes agents and operators will gravitate toward. Block them.

- **Computer Use for Missive or Gmail.** Both have full APIs. Clicking around your real desktop for email tasks is the wrong default.
- **Container browser for marketplace scraping.** Will get the Tenkara/SuperAgent IP blacklisted within days at any real volume. Use BrowserBase.
- **BrowserBase for your own Tenkara platform.** Adds latency, cost, and complexity for no anti-bot benefit. Use the Tenkara API or Container browser.
- **Browser Use when an API exists.** Browser flows are slower and more brittle than API calls. Always check for an API first, even if the web flow seems faster to wire up.
- **Skipping this decision because "the agent will figure it out."** It won't. An unguided agent picks the most visible tool, which is usually Computer Use or browser-based scraping — both wrong here.

## Tenkara automation mapping (quick lookup)

| Automation | Class | Mode |
|---|---|---|
| P0-A1 Supplier Leads Generator | Open web | **BrowserBase** |
| P0-A2 Material List Parser | Inbound email + parse | **Gmail/Missive API** |
| P0-A3 Supplier Legitimacy Checker | Open web + business registries | **BrowserBase** |
| P0-A4 Supplier Qualification Memory | Tenkara DB | **Tenkara API** |
| P0-A5 Marketplace Quote Scraper | Open web (marketplaces) | **BrowserBase** |
| P0-A6 Non-Marketplace Capability Scraper | Open web (supplier sites) | **BrowserBase** |
| P0-A7 Quote Extraction from email/PDF | Inbound email + parse | **Missive API + PDF parsing** |
| P0-A8 Grade Validator | Tenkara DB cross-check | **Tenkara API** |
| P1-A1 RFQ Builder | Templated email composition | **Missive API** |
| P1-A2 RFQ Tracker | Email metadata | **Missive API** |
| P1-A4 Document Intake (COA/Audit/Certs) | Email attachments + parse | **Missive API + PDF parsing** |
| P1-A5 Supplier Email Information Extraction | Inbound email + parse | **Missive API** |
| P1-A6 Certification Verification | External registries | **BrowserBase** (registries) **+ API** (Tenkara) |
| P1-A7 Quote Validation | Tenkara platform data | **Tenkara API** |
| P1-A8 Quote Expiry Alert | Tenkara DB + Slack notify | **Tenkara + Slack APIs** |
| P1-A11 Sample Request | Outbound email + platform | **Missive + Tenkara APIs** |
| P2-A1 PO Submission | Platform → email | **Tenkara + Missive APIs** |
| P2-A3 PO Validation | Inbound email vs DB | **Missive + Tenkara APIs** |
| P3-A1 Book Shipping for EXW | Carrier + platform | **Carrier + Tenkara APIs** |
| P3-A2 Send BOL to Supplier | Outbound platform → supplier | **Tenkara + Missive APIs** |
| P4-A3 Escalation Agent | Multi-system notification | **Missive + Slack + Tenkara APIs** |
| Email follow-up (E1-FU-NR etc.) | Missive draft + send | **Missive API** |
| Anything Computer Use | — | **None as of May 2026** |

## Output

When this skill is invoked, produce a single block in this exact format:

```
Automation: <name>
Target system(s): <list>
Recommended mode: <API/connector | Container browser | BrowserBase | Computer Use>
Reasoning: <one line, why this mode beats the alternatives>
Setup prerequisites: <API keys, BrowserBase project ID, connector OAuth, MCP, etc.>
Watchouts: <rate limits, ban risk, manual review needs, anything that needs a human in the loop>
```

No prose around it. Just the block.

## When to revisit this skill

This skill is opinionated for the Tenkara stack as of May 2026. Revisit if:

- SuperAgent adds a new browser host option beyond Container and BrowserBase.
- A target system you currently scrape via BrowserBase ships an official API — switch to the API.
- BrowserBase rate limits or pricing start binding on a real automation — evaluate proxy-based alternatives or hybrid scraping.
- You take on an automation involving a native desktop app for the first time — Computer Use will need real evaluation at that point, not just default avoidance.
