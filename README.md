# Warehouse REST API — Bruno OpenCollection Test Suite

Executable API test collection in [Bruno OpenCollection YAML (`v1.0.0`)](https://docs.usebruno.com/opencollection-yaml/overview) format for the **Warehouse REST API Testbench**.

---

## Repository Structure

```text
bruno_test_cases_creation/
├── opencollection.yml          # Collection root config, defaults, and fixture bootstrapper
├── environments/
│   └── local.yml               # Local environment configuration (base_url, credentials, seeded UUIDs)
├── auth/                       # Module: AUTH (Authentication & Role-Based Access Control)
│   ├── folder.yml
│   └── TC-AUTH-001..007*.yml
├── users/                      # Module: USER (User Governance & Account Lifecycle)
│   ├── folder.yml
│   └── TC-USER-001..017*.yml
├── catalog/                    # Module: PROD (Product Catalog & Warehouse Inventory Management)
│   ├── folder.yml
│   └── TC-PROD-001..013*.yml
└── orders/                     # Module: ORD (Transactional Orders & Lifecycle Orchestration)
    ├── folder.yml
    └── TC-ORD-001..015*.yml
```

---

## Test Coverage

| Module | Folder | Test Cases | Requests | Description |
| --- | --- | --- | --- | --- |
| **AUTH** | `auth/` | `TC-AUTH-001` – `TC-AUTH-007` | 10 | Admin & customer login, portal RBAC segregation, invalid credentials & constant-time dummy hash checks, empty credential validation, bcrypt 72-byte password boundary (`audit-f09`) |
| **USER** | `users/` | `TC-USER-001` – `TC-USER-017` | 32 | Commercial user creation, `409 USERNAME_TAKEN`, balance top-up & boundaries (`0`, negative, sub-cent, `1e18` overflow, `0.015` fractional), filter whitelists, whitespace trimming & empty string trap (`audit-f06`), customer profile, paginated directory, UUID validation, soft-deletion lifecycle, JWT revocation after soft-delete, username reuse prevention, double soft-delete, admin self-deletion guard (`audit-f04`), `VARCHAR(100)` username limit (`audit-f09`), admin RBAC |
| **PROD** | `catalog/` | `TC-PROD-001` – `TC-PROD-013` | 25 | Product registration, stock updates, negative price/stock validation, admin catalog, customer pagination & sorting, category (`userB`), brand (`userC`), and AND-intersection (`audit-f06`) whitelist enforcement, case-insensitive filters, pagination boundaries, blank/oversized model & float overflow price boundaries (`audit-f03`, `audit-f09`), non-paginated envelope hygiene (`audit-f07`), RBAC |
| **ORD** | `orders/` | `TC-ORD-001` – `TC-ORD-015` | 36 | Atomic order checkout (balance debit & stock decrement), `INSUFFICIENT_FUNDS` & `FAILED` ledger recording, `INSUFFICIENT_STOCK`, `FILTER_RESTRICTION`, quantity boundaries, immutable order history snapshots, cross-tenant `NOT_FOUND` isolation, admin order audit, order cancellation with exact-cent balance refund & inventory restocking, terminal status transition rejections, `NUMERIC(12,2)` overflow protection (`audit-f03`), exact-cent `$19.99 * 3 = $59.97` arithmetic precision, cancelling order of soft-deleted user (`audit-f04`), AND-intersection order blocking (`audit-f06`), multi-violation error precedence (`audit-f11`) |
---

## Prerequisites

1. **Warehouse REST API Testbench** running at `http://localhost:8080` (default) with seeded PostgreSQL database (`admin`, `userA`, `userB`, `userC`, and seeded catalog items).
2. **Bruno CLI (`bru`) v3.1.0+ / v4.x** installed:
   ```bash
   npm install -g @usebruno/cli
   bru --version
   ```

> **Note:** `opencollection.yml` includes a `before-request` fixture bootstrapper that automatically logs in `admin`, `userA`, `userB`, and `userC` and ensures seeded user filters, balances, and product stock levels before running any suite, folder, or individual request.

---

## Running Tests with `bru` CLI

Run all commands from the repository root directory:

### 1. Run the Full Test Suite
```bash
bru run -r --env local
```

### 2. Run by Domain Module (Folder)
```bash
# Authentication & Role-Based Access Control
bru run auth -r --env local

# User Governance & Account Lifecycle
bru run users -r --env local

# Product Catalog & Warehouse Inventory Management
bru run catalog -r --env local

# Transactional Orders & Lifecycle Orchestration
bru run orders -r --env local
```

### 3. Run Individual or Multi-Step Test Cases
```bash
# Run a single request
bru run users/TC-USER-001.yml --env local

# Run multi-step order creation + balance debit + stock decrement check
bru run \
  orders/TC-ORD-001-step1-capture-balance.yml \
  orders/TC-ORD-001-step2-submit-order.yml \
  orders/TC-ORD-001-step3-verify-balance-debited.yml \
  --env local

# Run multi-step order cancellation + balance refund + warehouse stock restoration check
bru run \
  orders/TC-ORD-009-step1-cancel-order.yml \
  orders/TC-ORD-009-step2-verify-balance-refunded.yml \
  orders/TC-ORD-009-step3-verify-stock-restored.yml \
  --env local
```

### 4. Filter by Tags
Available tags: `smoke`, `p0`, `auth`, `users`, `catalog`, `orders`, `admin`, `customer`, `rbac`, `security`, `bva`, `negative`, `pagination`, `whitelist`, `finance`, `inventory`, `lifecycle`, `cancellation`, `audit`, `isolation`, `transaction`, `core`, `conflict`, `validation`, `profile`, `filters`, `normalization`.

```bash
# Run smoke tests only
bru run -r --env local --tags smoke

# Run P0 priority tests
bru run -r --env local --tags p0

# Run RBAC and security tests
bru run -r --env local --tags rbac,security

# Exclude negative test cases
bru run -r --env local --exclude-tags negative
```

### 5. Override Environment Variables at Runtime
```bash
bru run -r --env local \
  --env-var base_url=http://localhost:8080 \
  --env-var admin_password=admin123 \
  --env-var user_a_password=user123
```

### 6. Generate Test Reports (JSON, JUnit XML, HTML)
```bash
bru run -r --env local \
  --reporter-json report.json \
  --reporter-junit report.xml \
  --reporter-html report.html
```
