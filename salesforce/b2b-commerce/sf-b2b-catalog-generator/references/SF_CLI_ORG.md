# Salesforce CLI — Org Selection

The skill requires picking a target Salesforce org when the catalog includes variations (Step 3b). This reference covers the exact commands and how to present the choice to the user.

## Prerequisites

- `sf` CLI installed (check with `which sf`).
- At least one org authenticated (`sf org login web` if none).
- The skill does **not** attempt to authenticate new orgs — if the user has no orgs, it reports that and stops before Step 3b.

## Step 1 — List connected orgs

```bash
sf org list --json 2>/dev/null
```

The JSON result has a `result` object with buckets:
- `nonScratchOrgs` — production, sandbox, and trial orgs
- `scratchOrgs` — scratch orgs
- `devHubs`
- `sandboxes`
- `other` — fallback bucket (often mirrors the others)

Orgs may appear in multiple buckets. Dedupe by `username` when presenting to the user.

Each org record has at minimum:
- `alias` — may be null; prefer this for display
- `username`
- `instanceUrl`
- `isDefaultUsername` — boolean, true for the current default
- `isSandbox`, `isScratch`, `isDevHub` — type flags

## Step 2 — Present to the user

Show a concise table, not the raw JSON. Example:

```
Connected orgs:
  1. Aliseda             storm.4d4223c6647c07@salesforce.com   storm-4d4223c6647c07.my.salesforce.com   (default)
  2. cursorTesting       storm.c0f6dbb3400646@salesforce.com   (other instance)
  3. JggDspMasterDemo    storm.c56fe1d140141e@salesforce.com   (other instance)

Which org should I use for this catalog? (1/2/3, or "skip" to only generate the CSV without deploying anything)
```

- **Always ask**, even with only one org connected, and even if there is a default. This protects against deploying to a production/customer org by mistake.
- Accept `skip` as a valid answer — if the user skips, drop variations from the catalog (see SKILL.md Step 3b).

## Step 3 — Confirm the selection

Once the user picks, run:

```bash
sf org display --target-org <alias> --json 2>/dev/null
```

and echo back:
- Alias
- Username
- Instance URL
- Org type (Production / Sandbox / Scratch / Trial / Demo)
- Connection status (should be `Connected`)

Then explicitly confirm before any deploy: "About to deploy metadata to **<alias>** (<instance URL>). Proceed?"

## Error cases

- **No orgs found** → stop, tell the user to run `sf org login web` and come back, or pick "skip" to generate only the CSV.
- **`sf` CLI not found** → tell the user it needs to be installed (`brew install --cask sf-cli` or the official installer); skip Step 3b.
- **Chosen org cannot be displayed** (expired token) → run `sf org login web --instance-url <url>` to refresh, or let the user pick another.

## What this reference does NOT do

- Does not create new orgs.
- Does not switch the default org — the chosen org is used for this session only via `--target-org <alias>` on every `sf` call.
- Does not deploy anything — that's in VARIATIONS_METADATA.md.
