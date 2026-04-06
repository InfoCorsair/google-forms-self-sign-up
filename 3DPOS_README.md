## Overview

This is a Google Apps Script (GAS) plugin for Google Sheets/Forms that enables self sign-up for Fabman member management. It is deployed via the `clasp` CLI and runs entirely within Google's cloud — there is no local build step or test runner.

## Deployment

```bash
clasp push   # Deploy all .js and appsscript.json to the linked Apps Script project
```

`clasp` must be authenticated and `.clasp.json` must exist locally (gitignored). There is no npm, no bundler, and no test framework.

## Code Structure

Three source files map directly to Apps Script modules — all globals are shared across files at runtime:

- **admin.js** — Installation (`onInstall`), menu setup (`onOpen`), and all sheet-creation/configuration logic. Creates and manages the Settings, Field Mappings, Package Mappings, Gender Mappings, and 3DPrinterOS sheets.
- **signup.js** — The `on_form_submitted` trigger handler. Reads form data, resolves field/package/gender mappings, calls the Fabman API to create a member, optionally creates a 3DPrinterOS account, and writes status back to the sheet.
- **shared.js** — All shared utilities: sheet accessors, configuration getters, Fabman API wrappers (`fetch`, `fetch_all`, `send_request`), 3DPrinterOS API helpers, and the `API_FIELDS` constant that maps human-readable field names to Fabman member object keys.

## Architecture

**Configuration is stored in Google Sheets**, not in code or environment variables:
- *Settings* sheet — Fabman API key, 3DPrinterOS credentials
- *Field Mappings* sheet — maps Google Form question titles to Fabman member fields
- *Package Mappings* sheet — maps form answer options to Fabman package IDs
- *Gender Mappings* sheet — maps form answer options to Fabman gender values

**Two installed triggers** (registered at install time, not simple triggers):
- `on_form_submitted` — fires on each form submission
- `on_installed_edit` — fires on sheet edits to keep mapping sheets in sync with the form

**Data flow on signup:**
1. `on_form_submitted` receives the form event
2. Each response item is looked up in the Field Mappings sheet via `get_field_map()`
3. `set_value()` handles special cases: date formatting, gender lookup, package name→ID resolution, concatenation when multiple fields map to the same Fabman field
4. Member is POSTed to Fabman API; on duplicate email a notification email is sent
5. If 3DPrinterOS is configured, a user account is created and logged to the 3DPrinterOS sheet

## External APIs

- **Fabman API**: `https://fabman.io/api/v1` — authenticated with Bearer token stored in the Settings sheet
- **3DPrinterOS API**: `https://cloud.3dprinteros.com/` — session-based auth (login → session token → create user)

Both domains are whitelisted in `appsscript.json` (`urlFetchWhitelist`). Any new external domain must be added there before `UrlFetchApp` will allow requests to it.

## Google Apps Script Constraints

- No `import`/`require` — all files share a single global scope at runtime
- `UrlFetchApp` replaces `fetch`; `SpreadsheetApp`, `FormApp`, `MailApp` replace their web equivalents
- OAuth scopes are declared in `appsscript.json` and cannot be changed at runtime
- Installed triggers (vs. simple triggers) are required for operations that need elevated permissions
