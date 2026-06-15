# Database Initialization

Postgres auto-loads any `.sql` files placed in this directory on first boot.

## Setup

Download the Northwind dataset:

```bash
curl -o northwind.sql https://raw.githubusercontent.com/pthom/northwind_psql/master/northwind.sql
```

The file is too large to commit to git (~342KB) and is sourced from the public `pthom/northwind_psql` repo.

## Schema

Tables loaded:
- customers (91 rows) — companies that buy from Northwind
- orders (830 rows) — each purchase made
- order_details (2,155 rows) — line items per order
- products (77 rows) — items sold
- categories (8 rows) — product types
- employees (9 rows) — staff handling orders
- suppliers (29 rows) — vendors
- shippers (3 rows) — delivery companies

Plus minor reference tables: region, territories, employee_territories, customer_demographics.
