# FinPulse Data Platform — Data Model (Phase 1)

## Overview
FinPulse simulates a retail banking data platform. Data originates from three
independent source systems, each with its own update cadence and quality
characteristics — mirroring how real banking data platforms ingest from
disparate, loosely-coupled systems rather than a single clean database.

## Source Systems

| Source System | Owns | Frequency | Notes |
|---|---|---|---|
| Core Banking System | Customers, Accounts | Daily batch | Legacy-style system, batch file drop |
| Card/Payment Processor | Transactions, Cards | Micro-batch (~15 min) | Near real-time feed |
| Branch Reference Data | Branches | Weekly / rarely | Static reference data |

## Entity Schemas

### Customers (Core Banking)
| Field | Type | Notes |
|---|---|---|
| customer_id | string (PK) | e.g. CUST00001 |
| first_name | string | |
| last_name | string | |
| date_of_birth | date | |
| email | string, nullable | |
| phone | string, nullable | |
| address_line | string | mutable — drives SCD |
| city | string | mutable |
| state | string | mutable |
| pincode | string | |
| kyc_status | enum: VERIFIED/PENDING/REJECTED | |
| customer_since_date | date | |
| last_updated_timestamp | timestamp | used for change detection |

### Accounts (Core Banking)
| Field | Type | Notes |
|---|---|---|
| account_id | string (PK) | e.g. ACC000001 |
| customer_id | string (FK -> Customers) | |
| account_type | enum: SAVINGS/CURRENT/LOAN | |
| branch_id | string (FK -> Branches) | |
| open_date | date | |
| status | enum: ACTIVE/DORMANT/CLOSED | |
| currency | string | e.g. INR |
| last_updated_timestamp | timestamp | |

### Transactions (Card/Payment Processor)
| Field | Type | Notes |
|---|---|---|
| transaction_id | string (PK) | |
| account_id | string (FK -> Accounts) | |
| transaction_timestamp | timestamp | when txn actually occurred |
| amount | decimal | |
| currency | string | |
| transaction_type | enum: DEBIT/CREDIT/TRANSFER | |
| channel | enum: ATM/POS/ONLINE/UPI/BRANCH | |
| merchant_name | string, nullable | only for POS/ONLINE |
| status | enum: SUCCESS/FAILED/PENDING | |
| ingestion_timestamp | timestamp | when WE received it; enables late-arrival detection |

### Cards (Card Processor)
| Field | Type | Notes |
|---|---|---|
| card_id | string (PK) | |
| account_id | string (FK -> Accounts) | |
| card_type | enum: DEBIT/CREDIT | |
| card_network | enum: VISA/MASTERCARD/RUPAY | |
| issue_date | date | |
| expiry_date | date | |
| status | enum: ACTIVE/BLOCKED/EXPIRED | |

### Branches (Reference Data)
| Field | Type | Notes |
|---|---|---|
| branch_id | string (PK) | |
| branch_name | string | |
| city | string | |
| state | string | |
| branch_type | enum: URBAN/RURAL/METRO | |

## Relationships
- Customers (1) → (N) Accounts
- Accounts (1) → (N) Transactions
- Accounts (1) → (N) Cards
- Branches (1) → (N) Accounts

Transactions is the eventual **fact table**; Customers, Accounts, Cards,
Branches become **dimension tables** in the Gold layer (star schema).

## Simulated Data Volume
| Entity | Volume |
|---|---|
| Customers | 5,000 |
| Accounts | ~7,500 |
| Cards | ~6,000 |
| Branches | 50 |
| Transactions | ~200,000, spread across 6 months of daily files |

## Intentional Data Quality Issues (injected by design)
These are deliberately introduced to simulate real-world source system
imperfections, and will be addressed in the Silver layer.

- **Customers**: ~1-2% duplicate records (slight name variations), nulls in
  email/phone, periodic address changes (SCD Type 2 candidate).
- **Transactions**: occasional duplicate transaction_ids (processor retries),
  late-arriving records (transaction_timestamp significantly earlier than
  ingestion_timestamp), invalid amounts (negative/zero), inconsistent
  currency codes.
- **Accounts/Cards**: occasional orphan records referencing not-yet-loaded
  parent records (out-of-order system loads).

## Ingestion Cadence Summary
- Customers, Accounts, Cards: daily batch
- Transactions: micro-batch (~15 min simulated)
- Branches: weekly / static

## Status
Phase 1 — Design finalized. Proceeding to Phase 1b: data generation scripts.