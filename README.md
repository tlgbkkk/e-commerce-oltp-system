# E - Commerce OLTP Faker System

A minimal e-commerce OLTP schema on PostgreSQL plus a Python script to generate fake data (Faker) and bulk-ingest it into the database.

## What’s inside

- `sql/init_tables`: Creates the schema (brand/category/seller/product/promotion/order/order_item/promotion_product) + a trigger that decreases stock on `order_item` inserts.
- `src/main.py`: Generates fake data and inserts it into PostgreSQL.
- `src/config.py`: DB connection config + seed + data volume knobs.
- `src/database.py`: `connect()` helper using `psycopg2`.

## Requirements

- PostgreSQL (tested with `psql 16.x`)
- Python `3.12`
- Poetry (tested with `Poetry 1.8.x`)

## Quickstart

### 1) Install dependencies

```bash
poetry install
```

### 2) Configure the database

Update `DB_CONFIG` in `src/config.py` for your environment.

Current defaults:

- host: `localhost`
- user: `postgres`
- password: `170723`
- database: `ecommerce_oltp`
- port: `5432`

### 3) Create the database + schema

Note: `sql/init_tables` contains `DROP TABLE IF EXISTS ...`, so it will remove existing data.

```bash
# create database (if needed)
psql -U postgres -c "CREATE DATABASE ecommerce_oltp;"

# create tables + trigger
psql -U postgres -d ecommerce_oltp -f sql/init_tables
```

### 4) Generate data and ingest into PostgreSQL

Run from the repo root:

```bash
poetry run python src/main.py
```

Without Poetry:

```bash
python3 src/main.py
```

## What gets inserted

`src/main.py` currently inserts:

- `brand`
- `category` (level 1 and level 2 with parent-child relationships)
- `seller`
- `promotion`
- `promotion_product`
- `product` (`discount_price` is computed from the best promotion for that product, with a 10% price floor)

Tables `order` and `order_item` exist in the schema, they will be used in next project.

## Tuning data volume

Edit in `src/config.py`:

- `SEED`: reproducibility.
- `DATA_VOLUME`: record counts per table.
- `CATEGORY_MAP`, `PROMO_NAMES`, `PROMO_TYPES`: sampling pools for random generation.

## Quick checks in SQL

```sql
-- Count rows in main tables
SELECT 'brand' AS t, COUNT(*) FROM brand
UNION ALL SELECT 'category', COUNT(*) FROM category
UNION ALL SELECT 'seller', COUNT(*) FROM seller
UNION ALL SELECT 'product', COUNT(*) FROM product
UNION ALL SELECT 'promotion', COUNT(*) FROM promotion
UNION ALL SELECT 'promotion_product', COUNT(*) FROM promotion_product;
```

Example: products joined with seller/brand/category:

```sql
SELECT
  p.product_id, p.product_name, p.price, p.discount_price, p.stock_qty,
  s.seller_name,
  b.brand_name,
  c.category_name
FROM product p
JOIN seller s ON s.seller_id = p.seller_id
JOIN brand b ON b.brand_id = p.brand_id
JOIN category c ON c.category_id = p.category_id
ORDER BY p.product_id
LIMIT 20;
```

## Troubleshooting

- `Database connection error: ...`: verify `src/config.py`, PostgreSQL is running, credentials are correct, and the database exists.
- `permission denied for database ...`: grant privileges or use a privileged user (for example `postgres`) when creating DB/schema.
- Slow runs: reduce `DATA_VOLUME["product"]` or scale up DB resources; the script uses bulk inserts and should be fast for moderate datasets.
