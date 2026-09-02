---
name: master-data
description: Manage QuickBooks master data — look up, create, or update accounts, vendors, customers, items, classes, and tax rates. Use when the user mentions chart of accounts, adding a vendor, creating a customer, new service item, tracking class, or updating any entity in QuickBooks.
---

# Master Data Skill

Manage the QuickBooks master data layer: retrieve IDs and details for all entity types, create new accounts/vendors/customers/items/classes, and update or deactivate existing records.

## Which company? (do this first)

One DeepLedger connection reaches every company the user can access. Before
anything else in this skill:

1. Call `qbCompanyProfile` (operation `profile`, the default). Its `company`
   field names the company you are in. Say the company name to the user in your
   first line.
2. If it answers `NO_ACTIVE_COMPANY`, call `qbCompanyProfile` with
   `operation: "list"`, then `operation: "switch"` with the `organizationId` the
   user means. Never guess between similar names; ask.
3. If the user names a different company than the active one, switch first.
   The switch is proven by a CompanyInfo read; a `qbConnected: false` answer
   means QuickBooks is not connected for that company and only tasks, documents,
   memory and the bank feed will work.
4. Every tool result carries `company`. If one carries `warning` (the active
   company was moved from the portal or another client), stop and confirm the
   company with the user before writing.

## Trigger

Activate when the user wants to:
- Look up an account, vendor, customer, item, class, or tax rate ID
- Add a new vendor or customer to QuickBooks
- Create a new account in the Chart of Accounts
- Add a new service, non-inventory, or inventory item
- Create or rename a class (department/location tracking)
- Update vendor details (address, 1099 status, tax ID)
- Update customer details (terms, address, contact info)
- Deactivate or merge duplicate entities
- Review Chart of Accounts structure

## Entity Types at a Glance

| Entity | Used For | Key Fields |
|--------|----------|------------|
| `account` | Chart of Accounts — categorize transactions | `accountType`, `name`, `accountNumber` |
| `vendor` | Payees on bills, expenses, 1099s | `name`, `address`, `vendor1099`, `taxIdentifier` |
| `customer` | Invoices, payments, sales receipts | `name`, `address` |
| `item` | Line items on invoices, bills, expenses | `itemType`, `incomeAccountId` / `assetAccountId` |
| `class` | Segment reporting (dept, location, project) | `name`, `parentClassId` |
| `taxRate` | Sales tax rates | Read-only; managed by QB Tax Center |

## Workflow: Retrieve Entity IDs

Use this before any transaction to get valid IDs.

1. **Call** — `qbMasterData(entityTypes=["account","vendor","customer","item","class"])` (request only what you need)
2. **Filter** — Pass `filter` to narrow by name when the list is long
3. **Disambiguate** — If multiple similar names exist, show the user the options and confirm which one before proceeding
4. **Never store mappings** — categorization truth is QB history fetched in real time (`qbFetchTransactions`); `agentMemory` holds only `policies`/`patterns`/`general` entries, never vendor→account or customer→item mappings (charter §8)

## Workflow: Get Vendor or Customer Details

When you need full contact, payment terms, or 1099 info:

1. **Call** — `qbMasterData(detailedInfo="vendor", filter="VendorName")` or `detailedInfo="customer"`
2. **Review** — Address, payment terms, 1099 eligibility, tax identifier, email, phone
3. **Use for** — Verifying 1099 setup, confirming billing address, checking payment terms before entering a bill

## Workflow: Create an Account

1. **Check for duplicates** — `qbMasterData(entityTypes=["account"])` and search by name before creating
2. **Determine type** — Confirm the correct `accountType` (see Account Types table below); this cannot be changed after creation
3. **Confirm** — Show the user: name, account type, account number (if any), parent account (if sub-account)
4. **Create** — `qbMasterData(operation="create", entityType="account", name, accountType, accountNumber?, parentAccountId?)`
5. **Verify** — Confirm the new account ID returned (no need to memorize it — `qbMasterData` lookups are realtime)

> `accountType` is **permanent** — it cannot be changed after the account is created. Always confirm with the user before creating.

## Workflow: Create a Vendor

1. **Check for duplicates** — `qbMasterData(entityTypes=["vendor"], filter="VendorName")` before creating
2. **Gather required info** — Name (display name in QB); optional: address, email, phone, payment terms
3. **1099 check** — Ask if vendor is an independent contractor who should receive a 1099; if yes, collect `taxIdentifier` (EIN or SSN) and set `vendor1099=true`
4. **Confirm** — Show: name, address, 1099 status, tax ID (masked)
5. **Create** — `qbMasterData(operation="create", entityType="vendor", name, address?, vendor1099?, taxIdentifier?)`

## Workflow: Create a Customer

1. **Check for duplicates** — `qbMasterData(entityTypes=["customer"], filter="CustomerName")` before creating
2. **Gather info** — Name, optional: billing address, email, phone, payment terms
3. **Confirm** — Show: name, address, contact details
4. **Create** — `qbMasterData(operation="create", entityType="customer", name, address?)`

## Workflow: Create an Item

Items appear on invoice/bill/expense line items. Item type determines required accounts.

1. **Check for duplicates** — `qbMasterData(entityTypes=["item"], filter="ItemName")` before creating
2. **Determine item type**:
   - `Service` — Labor, consulting, subscription fees → requires `incomeAccountId`
   - `NonInventory` — Physical goods not tracked by quantity → requires `incomeAccountId`
   - `Inventory` — Physical goods with quantity tracking → requires `assetAccountId`; QB auto-sets `TrackQtyOnHand=true`, `QtyOnHand=0`
3. **Lookup accounts** — `qbMasterData(entityTypes=["account"])` to get `incomeAccountId` or `assetAccountId`
4. **Confirm** — Show: item name, type, income/asset account it maps to
5. **Create** — `qbMasterData(operation="create", entityType="item", name, itemType, incomeAccountId? or assetAccountId?, expenseAccountId?)`

## Workflow: Create a Class

Classes add a segmentation dimension (department, location, project) to transactions.

1. **Check for duplicates** — `qbMasterData(entityTypes=["class"], filter="ClassName")`
2. **Determine hierarchy** — Is this a top-level class or sub-class? If sub-class, get the parent class ID first
3. **Confirm** — Show: name, parent class (if any)
4. **Create** — `qbMasterData(operation="create", entityType="class", name, parentClassId?)`

## Workflow: Update an Entity

1. **Retrieve current record** — `qbMasterData(detailedInfo="vendor"/"customer", filter="name")` to get `id` and `syncToken`
2. **Identify changes** — Present current values, confirm what should change
3. **Confirm** — Show before/after for changed fields
4. **Update** — `qbMasterData(operation="update", entityType, id, syncToken, ...changedFields)`
5. **Deactivate** — Pass `active=false` to deactivate (soft delete); the record is hidden but retained in history

> `syncToken` is required for all updates — it prevents overwriting concurrent changes. Always fetch it fresh before updating.

## Safety Checklist

- [ ] Duplicate check completed before creating any entity
- [ ] For accounts: `accountType` confirmed with user — cannot be changed after creation
- [ ] For 1099 vendors: `taxIdentifier` collected and `vendor1099=true` set
- [ ] For items: correct `itemType` and linked account confirmed
- [ ] `syncToken` fetched immediately before any update
- [ ] User confirmation shown before create or update
- [ ] No vendor→account or customer→item mappings written to `agentMemory` (QB history is the categorization truth — charter §8)

## Common Mistakes to Avoid

- Creating a duplicate vendor or customer — always search first with `filter`
- Setting the wrong `accountType` on a new account — it is permanent and cannot be changed
- Creating an `Income` account when the user needs an `Expense` account (or vice versa) — confirm the direction of money flow
- Forgetting `incomeAccountId` on Service/NonInventory items — QB requires it
- Forgetting `assetAccountId` on Inventory items — QB requires it
- Updating an entity without a fresh `syncToken` — the update will be rejected
- Using `taxRate` for create/update — tax rates are managed inside QB Tax Center only
- Deactivating a vendor or customer that still has open transactions — QB will warn; resolve outstanding items first
