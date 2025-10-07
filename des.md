# 📄 LLD: Bulk Group Creation Script (Spreadsheet Parser + Groups API Caller)

## 1. Overview

This script automates the **creation of groups** in the Access Management system by:
1. Reading a **spreadsheet file** where each **tab corresponds to a group**.
2. Parsing group metadata, resources, principals, and roles from each sheet.
3. Mapping the parsed data to the `GroupRequestDto`.
4. Calling the existing **Groups API** to create groups programmatically.

This approach avoids manual creation of hundreds of groups through the UI during initial plant setup.

---

## 2. Spreadsheet Structure

### 2.1 File Layout

- 📄 **Each worksheet/tab = One group**
- 🟡 **Tab name** = Group name
- 📝 Top rows contain group metadata
- 📊 Below rows contain resource-principal-role mappings

### 2.2 Tab Structure

| Row | Column A              | Column B                         | Column C          | Column D       |
|-----|-------------------------|-----------------------------------|-------------------|----------------|
| 1   | `<Group Name>`        | *(empty)*                       | *(empty)*        | *(empty)*    |
| 2   | `<Group Description>` | *(empty)*                       | *(empty)*        | *(empty)*    |
| 3   | Resource Type         | Resource IDs (comma separated) | Principals       | Roles        |
| 4+  | e.g. `machine`       | e.g. `M1, M2`                  | `user1@x.com`   | `role1`     |
| 5+  | e.g. `machine`       | `M3`                           | `user2@x.com`   | `role2`     |
| 6+  | e.g. `line`          | `L1, L2`                       | `user1@x.com`   | `role3`     |

---

## 3. DTO Mapping

### 3.1 DTO Recap

```ts
export class GroupRequestDto {
  name: string;
  description?: string;
  resource: GroupResourcesDto[];
  principals: GroupPrincipalDto[];
}

export class GroupResourcesDto {
  type: string;
  resourceIds: string[];
  cometApplicationIds?: string[];
}

export class GroupPrincipalDto {
  id: string;
  roleIds: string[];
}
