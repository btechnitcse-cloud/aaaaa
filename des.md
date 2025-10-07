# Webscript: Bulk Group Creation via Spreadsheet

**Goal:** Provide a simple, repeatable way to **bulk-create groups and assign roles** by parsing a spreadsheet and calling the existing **Access Management Groups API**. This is for **initial setup** (e.g., new manufacturing plants with 300–500 users) so they don’t need to click through the UI.

---

## 1) High-Level Design

- **Input**: Spreadsheet file (`.xlsx` preferred; `.csv` optional) filled by plant admins.
- **Process**: A TypeScript script (run locally from VS Code) parses the file, validates rows, expands them into group payloads, and **POSTs** to the Groups API.
- **Output**: Groups are created in Access Management; the script writes a **summary report** (success/failure per group) to `./out/report-<timestamp>.json` and `./out/report-<timestamp>.csv`.

### Why a local script (vs building a new API now)?
- No service deploys needed to iterate on parsing rules.
- Easy to run from VS Code with a launch config.
- Changes to templates/parsing are just a code change + re-run.
- We can still add a server-side API later if needed; this script becomes the reference implementation.

---

## 2) Repository Layout

```
webscript/
├─ src/
│  ├─ main.ts               # entrypoint – orchestrates the run
│  ├─ config.ts             # loads env/config JSON
│  ├─ spreadsheet/
│  │  ├─ parser.ts          # reads and validates spreadsheets
│  │  ├─ models.ts          # TS types for rows and domain
│  │  └─ validators.ts      # schema + row validation
│  ├─ api/
│  │  ├─ client.ts          # axios/fetch wrapper with retries
│  │  └─ groups.ts          # functions that call Groups API
│  ├─ util/
│  │  ├─ logger.ts          # structured logging
│  │  ├─ retry.ts           # retry with backoff, 429/5xx aware
│  │  └─ report.ts          # writes run summary
│  └─ runContext.ts         # holds per-run state (dry-run, etc.)
├─ templates/
│  ├─ bulk-groups.xlsx      # sample template (TODO populate)
│  └─ bulk-groups.csv       # CSV equivalent (optional)
├─ out/                     # generated reports
├─ .env.example
├─ config.example.json
├─ package.json
├─ tsconfig.json
├─ README.md                # this file
└─ .vscode/
   └─ launch.json
```

---

## 3) Spreadsheet Contract (Template)

We support two shapes. Choose **one** to keep parsing simple.

### Option A: **One row per group** (simplest for plants)
Each row represents a single group with comma-separated lists for resources, principals, and roles.

**Columns**

| Column | Required | Description |
|---|---|---|
| `GroupName` | ✅ | Group name (string) |
| `Description` | ❌ | Group description (string) |
| `ResourceTypes` | ✅ | Comma-separated list (e.g., `machine,line,area`) |
| `ResourceIds` | ✅ | Comma-separated list aligning with `ResourceTypes` (e.g., `M1,M2,M3`) |
| `ResourceValues` | ❌ | Optional, comma-separated list aligning with `ResourceTypes` |
| `Principals` | ✅ | Comma-separated list of user IDs or emails |
| `Roles` | ✅ | Comma-separated list of role IDs or role names |

> **Alignment rule:** `ResourceTypes`, `ResourceIds`, and `ResourceValues` (if present) must be **same length** per row and in the **same order**.

**Example row**

```
GroupName: Operators
Description: Shop floor operators
ResourceTypes: machine,machine,line
ResourceIds: M1,M2,LN-9
ResourceValues: , ,  # optional blanks allowed
Principals: alice@plant.com,bob@plant.com
Roles: role_read,role_execute
```

**Parsed payload (per row)**

```json
{
  "name": "Operators",
  "description": "Shop floor operators",
  "resources": [
    { "type": "machine", "id": "M1" },
    { "type": "machine", "id": "M2" },
    { "type": "line", "id": "LN-9" }
  ],
  "principals": ["alice@plant.com", "bob@plant.com"],
  "roles": ["role_read", "role_execute"],
  "commonAccountId": "<from config>"
}
```

### Option B: **One sheet per group** (tab = group)
- **Cell A1**: `GroupName`
- **Cell B1**: name value
- **Cell A2**: `Description`
- **Cell B2**: description value
- **From row 4 downwards**:
  - Columns: `ResourceType | ResourceId | ResourceValue (optional) | Principal | Role`
  - Multiple rows allowed; the union of all rows forms the group.

> This option is slightly harder to parse but friendlier to non-technical users. We can support both, but **start with Option A** for simplicity and add Option B later if plants ask for it.

---

## 4) API Contract (existing Groups API)

**Method**: `POST /groups` (URL base configured per environment)

**Body (assumed, align with AM service spec):**
```ts
type CreateGroupBody = {
  name: string;
  description?: string;
  resources: Array<{ type: string; id: string; value?: string }>;
  principals: string[]; // user identifiers (emails or IDs)
  roles: string[];      // role identifiers
  commonAccountId: string;
};
```

**Auth**: Bearer token (from `.env`/config), or service-to-service auth per environment.

---

## 5) Configuration

We support **two** config sources. Both can be used; env overrides JSON.

- **`.env`** (use `.env.example` as guide)  
  - `AM_BASE_URL=https://stage.access-mgmt.example.com`
  - `AM_TOKEN=...` (or client credentials fields)
  - `COMMON_ACCOUNT_ID=...`
  - `SPREADSHEET_PATH=./templates/bulk-groups.xlsx`
  - `ENV=stage`
  - `DRY_RUN=true`

- **`config.json`** (use `config.example.json`)  
  ```json
  {
    "amBaseUrl": "https://stage.access-mgmt.example.com",
    "auth": { "token": "…" },
    "commonAccountId": "account-123",
    "spreadsheetPath": "./templates/bulk-groups.xlsx",
    "dryRun": true,
    "concurrency": 4,
    "maxRetries": 5
  }
  ```

**Precedence**: `.env` > `config.json` > sane defaults.

---

## 6) Validation & Error Handling

- **Static schema**: Ensure required columns are present (Option A) or required cells (Option B).
- **Per-row validation**:
  - Non-empty `GroupName`.
  - `ResourceTypes` and `ResourceIds` lengths match.
  - `Principals` & `Roles` non-empty lists.
- **Soft warnings**: Trim whitespace, de-dupe entries.
- **Hard errors**: Row is skipped but recorded in report with details.
- **API errors**: Retries with backoff on 429/5xx; log response body for 4xx; continue with next group.
- **Idempotency** (optional): Add `--idempotent` flag to `GET /groups?name=` first, skip if exists.

---

## 7) Logging & Reporting

- **Console**: progress bar + structured logs (`logger.ts` with levels).
- **Report files**: JSON and CSV with per-group status:
  - `groupName`
  - `status` (created | skipped | failed)
  - `httpStatus` (if applicable)
  - `message` or `error`
  - `requestId` (if available from API response)

---

## 8) Concurrency & Rate Limiting

- Configurable **concurrency** (default 4).  
- **Retry** on 429/5xx with jittered exponential backoff.  
- Optional **sleep** between requests (e.g., `--perRequestDelayMs`).

---

## 9) Security

- Never commit real tokens. Use `.env` and add `.env` to `.gitignore`.
- If using client credentials, fetch token before run; avoid logging secrets.
- Support **dry-run** mode to print payloads without making API calls.

---

## 10) Developer Experience

### VS Code Launch Config (`.vscode/launch.json`)
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Run Webscript (dry-run)",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/node_modules/ts-node/dist/bin.js",
      "args": ["${workspaceFolder}/src/main.ts", "--dry-run"],
      "envFile": "${workspaceFolder}/.env"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Run Webscript",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/node_modules/ts-node/dist/bin.js",
      "args": ["${workspaceFolder}/src/main.ts"],
      "envFile": "${workspaceFolder}/.env"
    }
  ]
}
```

### `package.json` (scripts excerpt)
```json
{
  "name": "webscript-bulk-groups",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "start": "ts-node src/main.ts",
    "start:dry": "ts-node src/main.ts --dry-run",
    "lint": "eslint .",
    "test": "jest"
  },
  "dependencies": {
    "axios": "^1.7.0",
    "dotenv": "^16.4.0",
    "yargs": "^17.7.2",
    "xlsx": "^0.18.5",
    "p-limit": "^5.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.11.0",
    "ts-node": "^10.9.2",
    "typescript": "^5.4.0",
    "eslint": "^9.0.0",
    "jest": "^29.7.0",
    "@types/jest": "^29.5.12"
  }
}
```

---

## 11) Code Sketches

### `src/main.ts`
```ts
import { loadConfig } from './config';
import { parseSpreadsheet } from './spreadsheet/parser';
import { createGroups } from './api/groups';
import { createReport } from './util/report';
import { logger } from './util/logger';

async function main() {
  const cfg = await loadConfig(process.argv);
  logger.info({ cfg: { ...cfg, auth: '***' } }, 'Starting webscript');

  const rows = await parseSpreadsheet(cfg.spreadsheetPath, cfg.templateOption);
  logger.info({ count: rows.length }, 'Parsed groups');

  const results = await createGroups(rows, cfg);
  const reportPaths = await createReport(results);

  logger.info({ reportPaths }, 'Run complete');
}

main().catch((err) => {
  // ensure non-zero exit on fatal
  // eslint-disable-next-line no-console
  console.error(err);
  process.exit(1);
});
```

### `src/spreadsheet/models.ts`
```ts
export type GroupRow = {
  groupName: string;
  description?: string;
  resources: Array<{ type: string; id: string; value?: string }>;
  principals: string[];
  roles: string[];
};
```

### `src/api/groups.ts`
```ts
import { http } from './client';
import type { GroupRow } from '../spreadsheet/models';

export async function createGroups(rows: GroupRow[], cfg: any) {
  const results = [];
  for (const row of rows) {
    const body = {
      name: row.groupName,
      description: row.description,
      resources: row.resources,
      principals: row.principals,
      roles: row.roles,
      commonAccountId: cfg.commonAccountId,
    };

    if (cfg.dryRun) {
      results.push({ groupName: row.groupName, status: 'skipped', reason: 'dry-run' });
      continue;
    }

    try {
      const res = await http.post('/groups', body);
      results.push({ groupName: row.groupName, status: 'created', httpStatus: res.status, requestId: res.headers['x-request-id'] });
    } catch (e: any) {
      results.push({ groupName: row.groupName, status: 'failed', httpStatus: e?.response?.status, message: e?.response?.data || e.message });
    }
  }
  return results;
}
```

---

## 12) Testing Plan

- **Unit tests**: Parser validation (missing columns, misaligned lists), API client retry logic.
- **Dry-run**: Run against sample template; verify payloads printed and no network calls.
- **Stage smoke**: Point to stage AM, create 1–2 test groups, verify in UI & via GET API.
- **Edge cases**: Duplicates, blank rows, whitespace, very large principal lists.

---

## 13) Rollout & Usage

1. Share the **template** with the plant.
2. When returned, put the file at `SPREADSHEET_PATH` (or pass `--spreadsheet <path>`).
3. Set `COMMON_ACCOUNT_ID` and `AM_TOKEN` in `.env`.
4. Run:
   ```bash
   npm ci
   npm run start:dry    # preview
   npm run start        # execute
   ```
5. Inspect `out/report-*.json/csv`. Re-run if needed.

---

## 14) Future Enhancements

- Support **Option B (tab-per-group)** and auto-detection.
- Idempotent create (skip-if-exists) with `GET /groups?name=`.
- Parallel uploads with backpressure tuned by AM rate limits.
- Rich HTML/CSV report with direct links to created groups.
- Optional server-side API for self-serve uploads (Swagger).

---

## 15) Risks & Mitigations

- **Parsing mismatches** → Strict schema validation + dry-run preview.
- **Rate limiting** → Backoff retries + concurrency control.
- **Bad data** → Clear row-level errors in report; do not stop the whole run.
- **Secrets** → `.env` only, never commit.
- **Environment drift** → All env in config; report logs include `AM_BASE_URL` and `ENV`.

---

## 16) Acceptance Criteria (for the design ticket)

- A Markdown design in the repo (this file) that documents the approach.
- A working **dry-run** that parses the provided sample template and prints **valid payloads**.
- Configurable **COMMON_ACCOUNT_ID**, **AM_BASE_URL**, **token**, and **SPREADSHEET_PATH**.
- A generated **report** with per-group outcomes.
- Clear instructions in README for running from VS Code.

---

## 17) Quick Start Checklist

- [ ] `npm init -y && npm i axios dotenv xlsx yargs p-limit && npm i -D typescript ts-node @types/node`
- [ ] Add `tsconfig.json`, `package.json` scripts.
- [ ] Create `src/` files per structure.
- [ ] Copy `.env.example` and `config.example.json`; fill in local values.
- [ ] Place `templates/bulk-groups.xlsx`.
- [ ] Run `npm run start:dry`; verify payloads.
- [ ] Run `npm run start` against stage with 1–2 test groups.
- [ ] Share report + confirm groups exist in UI.
```

# End of file
