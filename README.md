# AIM: Renewal Terms

A single-page dashboard (Claude Artifact) that tracks whether Salesforce Renewal Opportunities are moving client contracts from single-year (<24 mo) to multi-year (>=24 mo) terms, or the reverse. It also shows account health, pricing & packaging status, AI maturity, and an "HRIS + AI-layer threat" score (Workday, Sana, TechWolf, Eightfold, Gloat, SAP AI).

- Live artifact: https://claude.ai/artifact/277jVjuxFKKKWEyEDdVZxq (owner: jdullaghan@degreed.com, shared with the Degreed org)
- Snapshot of this package: 2026-10-14

> **Data sensitivity:** the HTML and `db-backup/` contain real client names, ACV figures, account-health ratings and Gong/AIM account intelligence. Keep this repo private to people who already have access to that data in Salesforce.

## What is in this repo

| File | Purpose |
|---|---|
| `aim-renewal-terms.html` | The complete artifact, exactly as published (UI, logic, and a baked-in data snapshot). **This is the source of truth.** |
| `publish-config.json` | Title, icon, description and the **capabilities** (database + MCP connectors) the artifact must be published with. |
| `db-backup/snapshot/latest.json` | Copy of the artifact's shared database document `snapshot/latest` (the Oct 14, 2026 refresh, 136 rows). Newer than the data baked into the HTML (Aug 25, 2026). |
| `README.md` | This file. |

## How to recreate the artifact

1. Make sure you have the connectors enabled in claude.ai (Settings -> Connectors): **Degreed Internal MCP** (Salesforce) and **Google Drive**.
2. Open Claude Code (or claude.ai/code) in this repo and ask something like:
   > Publish `aim-renewal-terms.html` as an Artifact using the settings in `publish-config.json` (including the `capabilities` block). Then restore `db-backup/snapshot/latest.json` into the artifact database as collection `snapshot`, doc `latest`.
3. Claude will load the `artifact-design` and `artifact-capabilities` skills, publish with the `db` + `mcp` capabilities, and write the backup document with the `ArtifactData` tool (action `set`).
4. Open the page and click **Refresh from Salesforce + Drive**. This re-queries Salesforce and the Drive folders and rebuilds the data for the current fiscal window, so step 3's database restore is only a convenience (the page already shows the baked-in data without it).

Publishing from a different account creates a new URL; the old link stays with the original owner. If the owner leaves the company, **ask them (or an admin) to transfer ownership or republish from a team member's account before their access is removed**, otherwise the live link may stop working.

## How the page works (for maintainers)

Everything is in the one HTML file; no build step.

- **Baked data:** `BAKED_SNAPSHOT_AT`, `DATA` (opportunity rows), `OPP_ACCT` (opportunity -> account id map), `THREAT_BAKED` (threat inputs) and `DRIVE_ROOT` (Google Drive folder id of the AIM client folders) are constants in the `<script>`. They are only the starting point; Refresh replaces them.
- **Refresh (`refreshData`)**:
  1. Computes the fiscal window from today's date (`computeFiscalWindow`; FY runs Feb 1 - Jan 31; window = current fiscal quarter through three quarters out).
  2. Runs SOQL through the Salesforce MCP tool (`findQueryTool` / `runSoql`):
     - `Opportunity` where `Opportunity_Record_Type_Name__c = 'Renewals'` and `CloseDate` is in the window (also pulls Account fields `Pulse__c`, `New_Pricing_Packaging_Status__c`, `AI_Maturity__c`, `Degreed_AI_Exposure__c`, `AI_Maturity_Last_Assessed__c`, `AI_Maturity_Notes__c`, and `Owner.Manager.Name`).
     - `Contract` (original term from `ContractTerm`), `Quote` (primary or latest), and `QuoteLineItem` where `Recurring__c = true` (new term = earliest `Degreed_Start_Date__c` to latest `Degreed_End_Date__c`).
  3. Reloads threat inputs: Salesforce Account fields (`Primary_HRIS_HCM__c`, `Secondary_HRIS_HCM__c`, `Primary_LMS_SS__c`, `Competitive_Threat_Gong_AI__c`, `Account_Summary_Gong_AI__c`) plus the per-client Drive folders (`account.json`, `signals.json`, `aim_diagnostic.json`) matched by Salesforce Account Id.
  4. Writes the result to the shared database (`snapshot/latest`, and `threat/latest` for threat data) so every viewer sees it.
- **Scoring rules** (term classification, direction, threat tiers) are documented in the "Methodology & data notes" section at the bottom of the page and implemented in `classifyTerm`, `directionFor`, and `scoreThreat`.
- **Owner's manager buckets:** Carolyn Freeman, Devlin Janssens, Craig Hohls, Patricia Kramer, Jeff Smith, else "Other". Update this list in the code when org changes happen (search for `ownerManager`).

## Things to know / open items

- **Permissions for Refresh:** each viewer needs the Salesforce (Degreed Internal MCP) and Google Drive connectors added to their own claude.ai account. Without Drive, Refresh still works but keeps the last Drive threat snapshot.
- **Salesforce field names** are hard-coded in the SOQL. If admins rename or remove a custom field, Refresh will fail with a query error; fix the field name in `refreshData` and `refreshThreatLayers`.
- **`threat/latest` was empty** at hand-off, so the threat column shows the baked 2026-10-14 snapshot until someone runs Refresh.
- **Data-quality caveats** visible in the data: a few rows have unknown or inferred terms (`src` = `opportunity-field` or `unknown`); only ~35% of closed renewals org-wide link directly to a Contract, so older windows may need account/date inference.
- **Drive folder:** the AIM client folders are under Drive folder id in `DRIVE_ROOT`. The new owner needs access to that folder.
- The page does not call any external services other than the two MCP connectors and the artifact database.

## Updating this backup later

Re-run the export: read the live artifact with the Artifact tool (`action: "read"`) and save the HTML over `aim-renewal-terms.html`; export `snapshot/latest` with `ArtifactData` (`action: "get"`, `out_dir: "db-backup"`). Commit both.
