# Apache Ranger Guide for CDP (Cloudera Data Platform)

This guide walks through a hands-on Apache Ranger scenario built around a
fictional food-delivery dataset (`zomatodb`). It covers setting up the base
Hive dataset, creating role-based access policies for different user
personas, applying column masking to protect sensitive data, and enforcing
row-level filtering — followed by verification steps for each.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Prepare the Dataset in Hue](#step-1-prepare-the-dataset-in-hue)
4. [Step 2: Create Users for Each Persona](#step-2-create-users-for-each-persona)
5. [Step 3: Access Control Policies in Ranger](#step-3-access-control-policies-in-ranger)
6. [Step 4: Verify Access Policies](#step-4-verify-access-policies)
7. [Step 5: Column Masking](#step-5-column-masking)
8. [Step 6: Row-Level Filtering](#step-6-row-level-filtering)
9. [Policy Summary Table](#policy-summary-table)
10. [Key Concepts](#key-concepts)

---

## Overview

**Apache Ranger** is a centralized security framework for managing
fine-grained authorization across Hadoop ecosystem components (Hive, HDFS,
HBase, Kafka, etc.). Unlike HDFS's native POSIX/ACL model, Ranger lets
administrators define policies from a **single web UI** that apply at the
table, column, and even row level — including features like:

- **Access policies** — who can do what, on which database/table/column.
- **Masking policies** — hide or obscure sensitive column data for specific
  users, without altering the underlying data.
- **Row-level filters** — restrict which *rows* a user can see, based on a
  condition (e.g. only rows for a specific city).

This exercise models a food-delivery company (`zomatodb`) with three
business roles:

| Role | Represents | Needs access to |
|---|---|---|
| **Customer Support** | Support agents | Everything (for troubleshooting) |
| **Sales Team** | Sales analysts | Restaurant data only |
| **Delivery Boy** | Delivery staff | Customer contact info + order data |

---

## Prerequisites

- A CDP cluster with **Hue**, **Hive**, and **Ranger** installed and running.
- A Hive user account with permission to create databases/tables via Hue.
- Admin access to the **Ranger Admin UI**.
- A sample dataset (CSV or similar) representing customers, orders, and
  restaurants, ready to be loaded.

---

## Step 1: Prepare the Dataset in Hue

### 1.1 Log in to Hue

Log in to **Hue** using the **Hive** user account.

### 1.2 Upload the dataset

Place your dataset files into the HDFS directory:

```
/user/hive
```

This is typically done via Hue's File Browser (drag-and-drop upload) or:

```bash
hdfs dfs -put customer.csv order.csv restaurant.csv /user/hive/
```

### 1.3 Create the database

In Hue's **Hive Query Editor**, create the database:

```sql
CREATE DATABASE zomatodb;
```

### 1.4 Create the tables

Within `zomatodb`, create the three core tables:

```sql
USE zomatodb;

CREATE TABLE customer (
    customer_id INT,
    name        STRING,
    email       STRING,
    mobile      STRING,
    address     STRING,
    creditcard  STRING
);

CREATE TABLE `order` (
    order_id    INT,
    customer_id INT,
    restaurant_id INT,
    order_date  STRING,
    amount      DOUBLE
);

CREATE TABLE restaurant (
    restaurant_id INT,
    name          STRING,
    city          STRING,
    cuisine       STRING
);
```

> Adjust column names/types to match your actual dataset — the names above
> (`name`, `email`, `mobile`, `address`, `creditcard`, `city`, etc.) are
> referenced later when defining Ranger policies, so keep them consistent.

> **Note:** `order` is a reserved SQL keyword in Hive — wrap it in
> backticks (`` `order` ``) when creating/querying the table, as shown above.

---

## Step 2: Create Users for Each Persona

Create OS-level (or LDAP/AD-synced, depending on your cluster's identity
backend) user accounts representing each business role:

- `support` — Customer Support
- `sales` — Sales Team
- `delivery` — Delivery Boy

These usernames are what you'll reference when defining Ranger policies, so
make sure they match exactly what Ranger sees via its user sync (Unix sync,
LDAP sync, etc.).

---

## Step 3: Access Control Policies in Ranger

Navigate to the **Ranger Admin UI → Hive service (e.g. `cm_hive`) → Access →
Add New Policy** for each policy below.

### 3.1 Customer Support Policy — full access to everything

| Field | Value |
|---|---|
| Policy Name | `customer-support` |
| Hive Database | `zomatodb` |
| Table | `*` (all tables) |
| Columns | `*` (all columns) |
| User | `support` |
| Permissions | `all` |

**Purpose:** Support agents need to see full customer/order/restaurant
records to troubleshoot issues, so they get unrestricted access across the
entire database.

### 3.2 Sales Policy — restaurant data only

| Field | Value |
|---|---|
| Policy Name | `sales-policy` |
| Hive Database | `zomatodb` |
| Table | `restaurants` |
| Columns | `*` |
| User | `sales` |
| Permissions | `all` |

**Purpose:** Sales analysts only need restaurant-related data (for
onboarding, performance analysis, etc.) — no access to customer or order
tables is granted, so any query against those tables will be denied.

### 3.3 Delivery Policies — customer contact info + full order data

Delivery staff need **two separate policies**, since they need different
column-level access on different tables.

**Policy A — limited customer columns:**

| Field | Value |
|---|---|
| Policy Name | `delivery-policy-customers` |
| Hive Database | `zomatodb` |
| Table | `customer` |
| Columns | `name, email, mobile, address` |
| User | `delivery` |
| Permissions | `all` |

**Purpose:** Delivery staff need enough customer info to complete a
delivery (name, contact, address) — but **not** sensitive fields like
`creditcard`, which is deliberately excluded from the column list.

**Policy B — full order data:**

| Field | Value |
|---|---|
| Policy Name | `delivery-policy-orders` |
| Hive Database | `zomatodb` |
| Table | `order` |
| Columns | `*` |
| User | `delivery` |
| Permissions | `all` |

**Purpose:** Delivery staff need full visibility into order details (which
restaurant, what was ordered, delivery address linkage, etc.).

---

## Step 4: Verify Access Policies

For each user, log in (or `kinit`/query via Beeline/Hue as that user) and
confirm:

- **`support`** — can query `customer`, `order`, and `restaurant` tables,
  all columns, without restriction.
- **`sales`** — can query `restaurant` table only; queries against
  `customer` or `order` should be **denied**.
- **`delivery`** — can query `name, email, mobile, address` from `customer`
  (querying `creditcard` should be **denied** or return an error/empty
  result), and can query all columns of `order`.

```sql
-- As support: should succeed
SELECT * FROM zomatodb.customer;

-- As sales: should fail
SELECT * FROM zomatodb.customer;

-- As sales: should succeed
SELECT * FROM zomatodb.restaurant;

-- As delivery: should succeed
SELECT name, email, mobile, address FROM zomatodb.customer;

-- As delivery: should fail (creditcard not granted)
SELECT creditcard FROM zomatodb.customer;
```

---

## Step 5: Column Masking

Masking lets a user **query** a column but see an **obscured value**
instead of the real data — useful for sensitive fields like credit card
numbers, where support staff may need to confirm "yes, a card is on file"
without seeing the full number.

### Create the masking policy

Navigate to **Ranger Admin UI → Hive service → Masking → Add New Policy**.

| Field | Value |
|---|---|
| Policy Name | `Card-policy` |
| Hive Database | `zomatodb` |
| Hive Table | `customer` |
| Hive Column | `creditcard` |
| User | `support` |
| Access Type | `select` |
| Mask Type | `Show last 4` |

**Purpose:** `support` can already query the `customer` table (per the
`customer-support` policy), but instead of exposing the full
`creditcard` value, Ranger transparently rewrites the query result to show
only the last 4 digits (e.g. `XXXXXXXXXXXX1234`).

### Verify masking

Log in as `support` and query:

```sql
SELECT customer_id, name, creditcard FROM zomatodb.customer;
```

Expected result: `creditcard` values appear masked (only last 4 digits
visible), even though `support` has full `select` access to the column.

---

## Step 6: Row-Level Filtering

Row-level filters restrict **which rows** a query returns, based on a
condition — the user isn't blocked from the table or any column, but the
result set is transparently filtered.

### Create the row-level filter policy

Navigate to **Ranger Admin UI → Hive service → Row Level Filter → Add New
Policy**.

| Field | Value |
|---|---|
| Policy Name | `Bangalore-sales` |
| Hive Database | `zomatodb` |
| Hive Table | `restaurant` |
| User | `sales` |
| Access Type | `select` |
| Row Filter | `city='Bangalore'` |

**Purpose:** The `sales` user already has full access to the `restaurant`
table (per `sales-policy`), but this filter narrows their visibility to
**only rows where `city = 'Bangalore'`** — as if a hidden `WHERE` clause
were silently appended to every query they run against that table.

### Verify the row filter

Log in as `sales` and run:

```sql
SELECT * FROM zomatodb.restaurant;
```

Expected result: only rows where `city = 'Bangalore'` are returned, even
though no `WHERE` clause was written in the query — restaurants from other
cities are silently excluded.

---

## Policy Summary Table

| Policy Type | Policy Name | Table | Columns | User | Effect |
|---|---|---|---|---|---|
| Access | `customer-support` | `*` | `*` | `support` | Full access to all tables/columns |
| Access | `sales-policy` | `restaurants` | `*` | `sales` | Access limited to restaurant data |
| Access | `delivery-policy-customers` | `customer` | `name, email, mobile, address` | `delivery` | Limited customer contact fields only |
| Access | `delivery-policy-orders` | `order` | `*` | `delivery` | Full access to order data |
| Masking | `Card-policy` | `customer` | `creditcard` | `support` | Shows only last 4 digits |
| Row Filter | `Bangalore-sales` | `restaurant` | — | `sales` | Restricts visible rows to `city='Bangalore'` |

---

## Key Concepts

- **Access policies** answer *"can this user touch this table/column at
  all?"* — deny by default, explicit grant required.
- **Masking policies** answer *"can this user see the real value, or a
  transformed one?"* — the user still has access, but the data is altered
  in transit (e.g. partial redaction, hashing, nulling).
- **Row-level filters** answer *"which subset of rows can this user see?"*
  — access to the table/columns is unchanged, but a hidden filter condition
  is applied automatically.
- **Least privilege principle:** each persona in this exercise is granted
  only what their role requires — Sales never sees customer PII, Delivery
  never sees payment details, and even Support (who needs broad access) has
  the most sensitive field (`creditcard`) masked rather than fully exposed.
- **Policies stack, not replace:** a user can be affected by multiple policy
  types simultaneously (e.g. `support` has both an access policy *and* a
  masking policy on the same table) — Ranger evaluates all applicable
  policies together.
