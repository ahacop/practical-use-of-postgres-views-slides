---
title: Using Database Views in Rails
sub_title: A Practical Guide to PostgreSQL Views, Materialized Views, and Beyond
author: Ara Hacopian
theme:
  name: catppuccin-latte
---

# Introduction

Views are powerful database objects that can simplify complex queries, improve performance, and provide elegant abstractions in Rails applications.

Today we'll explore:

- Basic views in PostgreSQL
- Managing views with Scenic
- Materialized views
- Updatable views
- Incremental View Maintenance

<!-- end_slide -->

# Our Example Schema

First, let's define our example tables:

```sql {1-13|1-5|8-13|1-13}
CREATE TABLE orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_line_items (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id BIGINT NOT NULL REFERENCES orders(id),
  product_name TEXT NOT NULL,
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL
);
```

<!-- end_slide -->

# Table Query Results

**orders table:**

```
 id | customer_id |  status   |     created_at
----|-------------|-----------|-------------------
  1 |         101 | completed | 2024-01-15 10:30
  2 |         102 | pending   | 2024-01-16 14:20
  3 |         101 | completed | 2024-01-17 09:15
  4 |         103 | completed | 2024-01-18 16:45
```

<!-- end_slide -->

# Table Query Results

**order_line_items table:**

```
 id | order_id | product_name | quantity | unit_price
----|----------|--------------|----------|------------
  1 |        1 | Laptop       |        1 |     999.99
  2 |        1 | Mouse        |        2 |      29.99
  3 |        2 | Keyboard     |        1 |      79.99
  4 |        3 | Monitor      |        1 |     299.99
  5 |        3 | Cable        |        3 |       9.99
  6 |        4 | Headphones   |        1 |     149.99
```

<!-- end_slide -->

# What Are PostgreSQL Views?

A view is a **virtual table** based on the result of a SQL statement.

```sql {1-11|3-5,9|6-8,10|1-11}
CREATE VIEW order_summaries AS
SELECT
  orders.id,
  orders.customer_id,
  orders.status,
  COUNT(order_line_items.id) as item_count,
  SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount,
  AVG(order_line_items.unit_price) as avg_item_price
FROM orders
LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
GROUP BY orders.id, orders.customer_id, orders.status;
```

Views act like tables but don't store data themselves.

<!-- end_slide -->

# Querying Our View

Now we can query the view like any table:

```sql
SELECT * FROM order_summaries;
```

**Result:**

```
 id | customer_id |  status   | item_count | total   | avg_price
----|-------------|-----------|------------|---------|-----------
  1 |         101 | completed |          2 | 1059.97 |    514.99
  2 |         102 | pending   |          1 |   79.99 |     79.99
  3 |         101 | completed |          2 |  329.97 |    154.99
  4 |         103 | completed |          1 |  149.99 |    149.99
```

The complex join and aggregation logic is encapsulated in the view!

<!-- end_slide -->

# Using Views in Rails Models

Create a model for your view:

```ruby
class OrderSummary < ApplicationRecord
  self.primary_key = :id

  belongs_to :order

  scope :high_value, -> { where('total_amount > ?', 1000) }
  scope :multi_item, -> { where('item_count > ?', 1) }
  scope :completed, -> { where(status: 'completed') }

  def readonly? = true
end
```

Use it like any other model:

```ruby
OrderSummary.completed.high_value.multi_item
```

<!-- end_slide -->

# Views in Rails

```ruby
class CreateOrderSummariesView < ActiveRecord::Migration[8.0]
  def up
    execute <<~SQL
      CREATE VIEW order_summaries AS
      SELECT orders.id, orders.customer_id,
             SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount
      FROM orders LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
      GROUP BY orders.id, orders.customer_id;
    SQL
  end

  def down
    execute "DROP VIEW IF EXISTS order_summaries;"
  end
end
```

**Problems:** Hard to version, no rollbacks, difficult to maintain.

<!-- end_slide -->

# Scenic

Scenic is a Rails gem that makes managing database views simple and Rails-friendly.

```bash
gem install scenic
```

```ruby
# Gemfile
gem 'scenic'
```

Scenic provides:

- Version-controlled view definitions
- Rails migration integration
- Automatic rollbacks

<!-- end_slide -->

# Creating Your First View with Scenic

Generate a new view:

```bash
rails generate scenic:view order_summaries
```

This creates:

- Migration file: `db/migrate/xxx_create_order_summaries.rb`
- An empty SQL definition: `db/views/order_summaries_v01.sql`

<!-- end_slide -->

# Scenic Generated Migration

Scenic creates a clean, simple migration:

```ruby
# db/migrate/20250901123456_create_order_summaries.rb
class CreateOrderSummaries < ActiveRecord::Migration[8.0]
  def change
    create_view :order_summaries
  end
end
```

The migration automatically references the SQL file for the view definition.

<!-- end_slide -->

# Scenic Generated Migration

```sql
-- db/views/order_summaries_v01.sql
SELECT
  orders.id,
  orders.customer_id,
  orders.status,
  COUNT(order_line_items.id) as item_count,
  SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount,
  AVG(order_line_items.unit_price) as avg_item_price
FROM orders
LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
GROUP BY orders.id, orders.customer_id, orders.status;
```

<!-- end_slide -->

# Running the Migration

```bash
rails db:migrate
```

**Output:**

```
== 20250901123456 CreateOrderSummaries: migrating ===================
-- create_view(:order_summaries)
   -> 0.0045s
== 20250901123456 CreateOrderSummaries: migrated (0.0045s) ==========
```

That's it! Your view is now available in the database.

<!-- end_slide -->

# Updating Views with Scenic

Need to modify your view? Generate a new version:

```bash
rails generate scenic:view order_summaries
```

This creates `db/views/order_summaries_v02.sql` populated with the SQL definition in version one.

<!-- end_slide -->

# The Update Migration

```ruby
class UpdateOrderSummariesToVersion2 < ActiveRecord::Migration[8.0]
  def change
    update_view :order_summaries, version: 2, revert_to_version: 1
  end
end
```

Scenic automatically:

- Drops the old view
- Creates the new version
- Handles rollbacks to previous version

<!-- end_slide -->

# Materialized Views

- Regular views are computed **on every query**.

- Materialized views **store the result**.

- Must be manually refreshed

<!-- pause -->

```sql {1-12|1}
CREATE MATERIALIZED VIEW order_summaries_materialized AS
SELECT
  orders.id,
  orders.customer_id,
  orders.status,
  COUNT(order_line_items.id) as item_count,
  SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount,
  AVG(order_line_items.unit_price) as avg_item_price
FROM orders
LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
GROUP BY orders.id, orders.customer_id, orders.status;
```

Trade-off: **Performance vs. Data freshness**

<!-- end_slide -->

# When to Use Materialized Views

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Use When:**

- Complex, expensive queries
- Data doesn't change frequently
- Read-heavy workloads
- Reporting/analytics
- Query performance is critical

<!-- column: 1 -->

**Avoid When:**

- Data changes frequently
- Real-time accuracy required
- Simple queries
- Write-heavy workloads
- Storage is constrained

<!-- reset_layout -->

<!-- end_slide -->

# Materialized Views with Scenic

```bash
rails generate scenic:view order_summaries_materialized --materialized
```

```sql
-- db/views/order_summaries_materialized_v01.sql
SELECT
  orders.id,
  orders.customer_id,
  orders.status,
  COUNT(order_line_items.id) as item_count,
  SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount,
  AVG(order_line_items.unit_price) as avg_item_price
FROM orders
LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
GROUP BY orders.id, orders.customer_id, orders.status;
```

<!-- end_slide -->

# Refreshing Materialized Views

Materialized views need to be refreshed when underlying data changes.

Scenic provides a built-in helper:

```ruby
class OrderSummaryMaterialized < ApplicationRecord
  def self.refresh!
    Scenic.database.refresh_materialized_view(table_name)
  end
end
```

Under the hood, this calls:

```sql
REFRESH MATERIALIZED VIEW order_summaries_materialized;
```

<!-- end_slide -->

# Scheduling Materialized View Refreshes

Use background jobs to refresh periodically:

```ruby
# config/queue.yml
production:
  refresh_materialized_views:
    command: "OrderSummaryMaterialized.refresh!"
    schedule: every hour
```

<!-- end_slide -->

# Simple Updatable Views

Some PostgreSQL views are automatically updatable:

**Requirements:**

- Single table (no joins)
- No aggregate functions
- No DISTINCT, GROUP BY, HAVING
- No set operations (UNION, INTERSECT)

```sql
-- This view is automatically updatable
CREATE VIEW pending_orders AS
SELECT id, customer_id, status, created_at
FROM orders
WHERE status = 'pending';
```

<!-- end_slide -->

# Using Simple Updatable Views

```ruby
class PendingOrder < ApplicationRecord
  # ...
end

# These operations work:
order = PendingOrder.find(1)
order.update(status: "processing")  # ✅ Works
order.destroy                       # ✅ Works

PendingOrder.create(customer_id: 104, status: "pending")  # ✅ Works
```

The updates are applied to the underlying `orders` table automatically.

<!-- end_slide -->

# Non-Updatable Views

Even simple join views aren't automatically updatable:

```sql {2-6|8-13}
-- Example: users and profiles tables
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50),
  email VARCHAR(100)
);

CREATE TABLE user_profiles (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  first_name VARCHAR(50),
  last_name VARCHAR(50)
);
```

<!-- end_slide -->

# Non-Updatable Views

```sql
-- This view is NOT updatable
CREATE VIEW user_details AS
SELECT u.username, u.email, p.first_name, p.last_name
FROM users u
JOIN user_profiles p ON u.id = p.user_id;
```

<!-- end_slide -->

# Making Views Updatable with Triggers

Use **INSTEAD OF triggers** to handle inserts:

```sql
CREATE TRIGGER user_details_insert
  INSTEAD OF INSERT ON user_details
  FOR EACH ROW EXECUTE FUNCTION insert_user_details();
```

<!-- pause -->

The same applies to updates and deletes.

<!-- end_slide -->

# Making Views Updatable with Triggers

```sql {1|2|3-4|6-10|12-14}
CREATE OR REPLACE FUNCTION insert_user_details()
RETURNS TRIGGER AS $$
DECLARE
  new_user_id INTEGER;
BEGIN
  -- Insert into users table
  INSERT INTO users (username, email)
  VALUES (NEW.username, NEW.email)
  RETURNING id INTO new_user_id;

  -- Insert into user_profiles table
  INSERT INTO user_profiles (user_id, first_name, last_name)
  VALUES (new_user_id, NEW.first_name, NEW.last_name);

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

<!-- end_slide -->

# Making Views Updatable with Triggers

```sql
-- Now this works!
INSERT INTO user_details (username, email, first_name, last_name)
VALUES ('alice', 'alice@example.com', 'Alice', 'Johnson');

SELECT * FROM user_details;
```

**Result:**

```
username | email             | first_name | last_name
---------|-------------------|------------|----------
alice    | alice@example.com | Alice      | Johnson
```

The trigger automatically created records in both underlying tables.

<!-- end_slide -->

# Managing Triggers with fx Gem

The `fx` gem helps manage PostgreSQL functions and triggers in Rails:

```ruby
# Gemfile
gem 'fx'
```

Generate function and trigger:

```bash
rails generate fx:function insert_user_details
rails generate fx:trigger user_details_insert
```

This creates versioned SQL files similar to Scenic views.

<!-- end_slide -->

# Materialized Views -- Caveats

- `REFRESH MATERIALIZED VIEW` recalculates everything.

<!-- end_slide -->

# Incrementally Maintained Materialized Views

Customer Rollup Table

```sql {1-20|4|5|6-8}
CREATE MATERIALIZED VIEW customer_order_stats AS
SELECT
  customer_id,
  COUNT(*) as total_orders,
  COUNT(*) FILTER (WHERE status = 'completed') as completed_orders,
  SUM((SELECT SUM(quantity * unit_price)
       FROM order_line_items
       WHERE order_id = orders.id)) FILTER (WHERE status = 'completed') as total_spent
FROM orders
GROUP BY customer_id;

-- Add unique index for efficient updates
CREATE UNIQUE INDEX ON customer_order_stats (customer_id);
```

<!-- end_slide -->

# Incrementally Maintained Materialized Views

Trigger to Update Customer Stats

```sql
-- Attach trigger to orders table
CREATE TRIGGER customer_stats_refresh
  AFTER INSERT OR UPDATE OR DELETE ON orders
  FOR EACH ROW EXECUTE FUNCTION refresh_customer_stats();
```

<!-- end_slide -->

# Incrementally Maintained Materialized Views

Trigger to Update Customer Stats (simplified example)

```sql {4|6|7-11}
CREATE OR REPLACE FUNCTION refresh_customer_stats()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.status != 'completed' AND NEW.status = 'completed' THEN
    UPDATE customer_order_stats
    SET completed_orders = completed_orders + 1,
        total_spent = total_spent + (
          SELECT SUM(quantity * unit_price)
          FROM order_line_items
          WHERE order_id = NEW.id
        )
    WHERE customer_id = NEW.customer_id;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

<!-- end_slide -->

# Modern IVM Solutions

**Incremental View Maintenance (IVM)** automatically updates materialized views:

**pg_ivm Extension:**

- PostgreSQL extension for automatic IVM
- Maintains materialized views incrementally
- Supports simple aggregations and joins
- Still in development
- Available on Neon.

<!-- end_slide -->

# pg_ivm Extension

```sql
-- Install pg_ivm extension
CREATE EXTENSION pg_ivm;

-- Create incrementally maintained view
SELECT create_immv('order_summaries',
  'SELECT orders.id, orders.customer_id,
          COUNT(order_line_items.id) as item_count,
          SUM(order_line_items.quantity * order_line_items.unit_price) as total_amount
   FROM orders LEFT JOIN order_line_items ON orders.id = order_line_items.order_id
   GROUP BY orders.id, orders.customer_id');
```

<!-- end_slide -->

# Other IVM Tools

- **Materialize**: Streaming SQL database with real-time views
- **ClickHouse**: Materialized views with real-time updates
- **Feldera**: Query engine for incremental computation
- **TimescaleDB**: Time-series data with continuous aggregates

<!-- end_slide -->

# Choosing the Right Approach

<!-- column_layout: [1, 1] -->

<!-- column: 0 -->

**Regular Views:**

- Need Real-time data
- Query is fast enough, indexes sufficient

**Materialized Views:**

- Complex aggregations
- Would be too expensive if not cached
- High query frequency
- Acceptable data lag
- ✅ Use Scenic + refresh jobs

<!-- column: 1 -->

**Incremental Updates:**

- Large datasets
- Frequent data changes
- Performance critical
- ✅ Custom triggers or pg_ivm

<!-- reset_layout -->

<!-- end_slide -->

# Questions?

Thank you!

**Resources:**

- Scenic: github.com/scenic-views/scenic
- fx: github.com/teoljungberg/fx
- pg_ivm: github.com/sraoss/pg_ivm
- PostgreSQL Documentation: postgresql.org/docs/

**Contact:**

- GitHub: @ahacop
- Bsky: @hacopian.de
- Email: <ara@hacopian.de>
