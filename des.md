# 📄 Bulk Group Creation from Spreadsheet — Design Document

This document outlines the design for a script to support **bulk creation of groups** in Access Management by parsing a **structured spreadsheet** and calling the **Groups API**.  
This feature will enable fast initial access setup (e.g., onboarding 300–500 users) without manual UI operations.

---

## 📎 Relevant Links
- **Ticket:** [ACM-XXX](#)  
- **MR:** [link](#)  
- **Reference Conversation:** Internal discussion on spreadsheet vs API based approach (Sanjay & team, Oct 2025).  
- **Access Management API:** `POST /groups`

---

## 📊 Spreadsheet Template Schema

The input is a **spreadsheet file (`.xlsx`)**, where **each sheet/tab represents a single group**.  
This file is provided by **plant administrators**, filled manually, and uploaded/executed locally.

### Sheet Structure

| Row | Column A              | Column B                         | Column C          | Column D       |
|-----|-------------------------|-----------------------------------|-------------------|----------------|
| 1   | `<Group Name>`        | *(empty)*                       | *(empty)*        | *(empty)*    |
| 2   | `<Group Description>` | *(empty)*                       | *(empty)*        | *(empty)*    |
| 3   | Resource Type         | Resource IDs (comma separated) | Principals       | Roles        |
| 4+  | e.g. `machine`       | e.g. `M1, M2`                  | `user1@x.com`   | `role1`     |
| 5+  | e.g. `machine`       | `M3`                           | `user2@x.com`   | `role2`     |
| 6+  | e.g. `line`          | `L1, L2`                       | `user1@x.com`   | `role3`     |

- **Tab Name** = Group Name (fallback if A1 missing)
- **A2** = Group Description
- **Rows 4+** = Table defining **resources**, **principals**, and **roles**

---

## 🧠 DTO Mapping

The spreadsheet maps directly to the existing DTO used by the Groups API:

### `GroupRequestDto`
```ts
{
  "name": "string",
  "description": "string",
  "resource": GroupResourcesDto[],
  "principals": GroupPrincipalDto[]
}
```

### Mapping Table

| Spreadsheet Data | DTO Field |
|-------------------|-----------|
| Tab name / A1    | `name` |
| A2               | `description` |
| Column A (resource type) | `resource[].type` |
| Column B (resource IDs) | `resource[].resourceIds` (comma split) |
| Column C (principal) | `principals[].id` |
| Column D (roles) | `principals[].roleIds` (comma split) |

---

## 🧾 Sample Spreadsheet to DTO Conversion

**Input (sheet “Operators”)**:

| A          | B           | C                    | D                |
|------------|-------------|-----------------------|-------------------|
| Operators  |             |                       |                   |
| Shop Floor |             |                       |                   |
| machine   | M1, M2     | alice@example.com   | role-operator  |
| machine   | M3         | bob@example.com     | role-maint    |
| line      | L1, L2     | alice@example.com   | role-supervisor |

**Output DTO:**

```json
{
  "name": "Operators",
  "description": "Shop Floor",
  "resource": [
    { "type": "machine", "resourceIds": ["M1", "M2", "M3"] },
    { "type": "line", "resourceIds": ["L1", "L2"] }
  ],
  "principals": [
    { "id": "alice@example.com", "roleIds": ["role-operator", "role-supervisor"] },
    { "id": "bob@example.com", "roleIds": ["role-maint"] }
  ]
}
```

---

## 🧪 Script Overview

A **Node.js + TypeScript** script will be created in a separate GitLab project.  
It will:
1. Parse spreadsheet file (`xlsx`)  
2. Convert each sheet into a `GroupRequestDto` object  
3. Call `POST /groups` API for each group  
4. Log results and generate a **JSON/CSV report**

### Why a Script (not API)?
- Fast to build & iterate — no deployments required  
- Flexible for different spreadsheets per plant  
- Easier to maintain parsing logic locally  
- Potential to evolve into an API later if needed

---

## ⚡ CLI Interface

The script accepts **spreadsheet file path** and environment configs:

```bash
ts-node src/main.ts --spreadsheet ./data/plant1.xlsx --env stage
```

Or via `.env`:

```
SPREADSHEET_PATH=./data/plant1.xlsx
AM_BASE_URL=https://stage.access-mgmt.com
COMMON_ACCOUNT_ID=xxx
AM_TOKEN=xxxxx
```

---

## 🧰 Tech Stack

| Purpose              | Tool |
|-----------------------|------|
| Spreadsheet parsing   | `xlsx` |
| HTTP API calls        | `axios` |
| CLI args              | `yargs` |
| Config                | `dotenv` |
| Concurrency control   | `p-limit` |
| Logging               | `chalk` (optional) |

No Electron, no UI.

---

## 🧭 Flow

1. **Parse CLI/Env Config**  
2. **Read Spreadsheet (xlsx)**  
3. For each **sheet/tab**:
   - Extract group name/desc (rows 1–2)
   - Build resource/principal structures (rows 4+)
   - Validate DTO fields
   - Push into queue
4. **POST /groups** for each group object  
5. Collect results → Write JSON/CSV report

---

## 🛡️ Validation Rules

| Field | Rule |
|-------|-----|
| Group Name | Non-empty |
| Resource Type | Non-empty |
| Resource IDs | At least one |
| Principal ID | Must be valid email |
| Role IDs | At least one per principal |

- Rows with missing critical data → skipped but logged  
- Email & role validation done before API call

---

## 🧱 Error Handling

- **Spreadsheet errors** → logged with row number
- **Validation errors** → skipped group or principal, continue others
- **API errors**:
  - 4xx → logged, skipped
  - 5xx or 429 → retried with exponential backoff (max 3)
- Failed groups → included in report

---

## 📝 Report File

Generated after each run:

```json
[
  { "groupName": "Operators", "status": "created", "httpStatus": 201 },
  { "groupName": "QA-Team", "status": "failed", "httpStatus": 400, "error": "Invalid role ID" }
]
```

Also saved as CSV for easy sharing.

---

## 📅 Tickets to be Created

- [ ] Create new GitLab project `access-mgmt-bulk-group-script`
- [ ] Implement spreadsheet parser
- [ ] Implement DTO mapper
- [ ] Implement Groups API caller with retries
- [ ] Implement CLI config + `.env` loading
- [ ] Add report generation
- [ ] Add validation & error handling
- [ ] Optional: add dry-run mode

---

## 📌 Notes

- This tool will be used for **one-time initial group setup** per plant.
- Later, this could evolve into a server-side endpoint to accept spreadsheets via Swagger.
- CometApplicationIds column can be added in future if required.

---

## 🟢 Acceptance Criteria

- ✅ Spreadsheet parsing matches defined structure  
- ✅ Validations implemented for required fields  
- ✅ Groups API is called with correct DTO payload  
- ✅ Script logs successes/failures clearly  
- ✅ Report file generated with per-group outcomes  
- ✅ Supports CLI args + `.env` config  
- ✅ Handles multiple sheets in one file

---
