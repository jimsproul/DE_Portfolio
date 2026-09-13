Here is a GitHub README-ready Markdown section:

````markdown
## Dimensional Data Model

### Overview

A **dimensional data model** organizes data specifically for analytics, reporting, and business intelligence workloads. Unlike a normalized relational model, which focuses on reducing duplication and maintaining transactional integrity, a dimensional model is optimized to make analytical queries **simple, understandable, and efficient**.

The model is generally organized as a **star schema** consisting of:

- A central **fact table** containing business events and measurable values.
- Multiple **dimension tables** containing descriptive information used to filter, group, and categorize those events.

For the Northwind dataset, the primary business event is a **product being sold as part of an order**.

The dimensional model can therefore be visualized conceptually as:

```text
                         Dim_Date
                            │
                            │
Dim_Customer ──────── Fact_Orders ──────── Dim_Employee
                       /    |    \
                      /     |     \
             Dim_Product    |    Dim_Territory
                            |
                      Dim_Category
```

---

## Fact Table

### Fact_Orders

`Fact_Orders` is the central fact table.

Although the source is based on `orders.csv`, the useful analytical grain requires information from both:

```text
orders.csv
     +
order-details.csv
     ↓
Fact_Orders
```

The **grain** of the fact table is:

> **One row per product line item on an order.**

For example, if order `10248` contains three different products, `Fact_Orders` contains three rows for that order.

A simplified structure might be:

```text
Fact_Orders
├── order_line_key        PK
├── order_id
├── order_date_key        FK
├── customer_key          FK
├── employee_key          FK
├── product_key           FK
├── category_key          FK
├── territory_key         FK
├── quantity
├── unit_price
├── discount
└── extended_amount
```

The fact table contains two general categories of columns:

### Dimension Keys

Foreign keys connect the business event to its descriptive dimensions:

```text
customer_key
employee_key
product_key
category_key
territory_key
order_date_key
```

These answer questions such as:

- Who purchased the product?
- Which employee handled the sale?
- What product was sold?
- What category did the product belong to?
- Where did the sale occur?
- When did the transaction occur?

### Measures

Numeric values describe the transaction itself:

```text
quantity
unit_price
discount
extended_amount
```

For example, an analytical measure could be calculated as:

```text
extended_amount =
    quantity × unit_price × (1 - discount)
```

These measures can then be aggregated using operations such as:

```text
SUM()
AVG()
MIN()
MAX()
COUNT()
```

---

## Dimension Tables

### Dim_Customer

`Dim_Customer` describes the customer associated with each sale.

It originates primarily from:

```text
customers.csv
```

Typical attributes include:

```text
Dim_Customer
├── customer_key         PK
├── customer_id
├── company_name
├── contact_name
├── contact_title
├── address
├── city
├── region
├── postal_code
├── country
├── phone
└── fax
```

This dimension enables analysis such as:

- Sales by customer
- Sales by company
- Sales by city
- Sales by region
- Sales by country

The relationship is:

```text
Dim_Customer 1 ─────< N Fact_Orders
```

One customer can therefore be associated with many sales transactions.

---

### Dim_Product

`Dim_Product` describes the products being sold.

It originates from:

```text
products.csv
```

Typical attributes include:

```text
Dim_Product
├── product_key          PK
├── product_id
├── product_name
├── unit_price
├── units_in_stock
└── discontinued
```

The relationship is:

```text
Dim_Product 1 ─────< N Fact_Orders
```

A product can appear in thousands of fact-table records, but each fact record references one product.

This supports questions such as:

```text
Which products generate the most revenue?

What is the average quantity sold by product?

Which products are declining in sales?

How much revenue comes from discontinued products?
```

---

### Dim_Category

`Dim_Category` describes product categories.

It originates from:

```text
categories.csv
```

Typical attributes include:

```text
Dim_Category
├── category_key         PK
├── category_id
├── category_name
└── description
```

The relationship is:

```text
Dim_Category 1 ─────< N Fact_Orders
```

This allows transactions to be aggregated at a higher business level than individual products.

For example:

```text
Revenue by Category
Quantity Sold by Category
Average Order Value by Category
```

In a more aggressively denormalized star schema, category attributes could instead be incorporated directly into `Dim_Product`. Keeping `Dim_Category` separate introduces a small degree of **snowflaking** or an additional dimensional relationship.

---

### Dim_Employee

`Dim_Employee` describes the employee responsible for an order.

It originates from:

```text
employees.csv
```

Typical attributes include:

```text
Dim_Employee
├── employee_key         PK
├── employee_id
├── first_name
├── last_name
├── title
├── hire_date
├── reports_to
├── city
├── region
└── country
```

The relationship is:

```text
Dim_Employee 1 ─────< N Fact_Orders
```

This enables analysis such as:

```text
Revenue by Employee
Orders by Employee
Average Order Value by Employee
Sales by Employee Region
```

---

### Dim_Territory

`Dim_Territory` represents sales territories.

It originates primarily from:

```text
territories.csv
```

Typical attributes include:

```text
Dim_Territory
├── territory_key          PK
├── territory_id
├── territory_description
└── region_id
```

The analytical relationship is:

```text
Dim_Territory 1 ─────< N Fact_Orders
```

This makes it possible to analyze measures geographically:

```text
Revenue by Territory
Orders by Territory
Products Sold by Territory
Employee Performance by Territory
```

---

## Employee-to-Territory Relationship

The original relational model contains:

```text
employees.csv
        │
        │
employee-territories.csv
        │
        │
territories.csv
```

`employee-territories.csv` resolves a many-to-many relationship:

```text
Employee N >─────< N Territory
```

The same relationship could be represented in a graph database as:

```text
(:Employee)-[:SELLS_IN]->(:Territory)
```

This relationship requires special consideration when building the dimensional model.

If a transaction can be assigned unambiguously to a single sales territory, the resulting fact row can contain:

```text
territory_key
```

and the model becomes:

```text
Dim_Territory 1 ─────< N Fact_Orders
```

However, if an employee can simultaneously belong to multiple territories and an individual order does **not** identify which territory generated the sale, assigning a territory directly to the fact table would be ambiguous.

In a production dimensional model, this may require a **bridge table**:

```text
Dim_Employee
     │
     │ 1
     │
     N
Bridge_Employee_Territory
     N
     │
     │ 1
     │
Dim_Territory
```

This preserves the original many-to-many relationship without incorrectly attributing a transaction to a territory.

---

## Dim_Date

`Dim_Date` is a dimension created specifically for analytical processing rather than being supplied directly by the Northwind CSV files.

A typical date dimension contains:

```text
Dim_Date
├── date_key             PK
├── full_date
├── year
├── quarter
├── month
├── month_name
├── week_of_year
├── day
├── day_name
└── is_weekend
```

The relationship is:

```text
Dim_Date 1 ─────< N Fact_Orders
```

The date dimension makes time-based analysis considerably easier.

For example:

```text
Sales by Year
Sales by Quarter
Sales by Month
Week-over-Week Sales
Year-over-Year Sales
Weekend vs. Weekday Sales
```

The same date dimension can also be reused for different dates contained in an order, such as:

```text
order_date
required_date
shipped_date
```

These are known as **role-playing dimensions**. A single `Dim_Date` can logically serve as:

```text
             ┌── Order Date
Dim_Date ────┼── Required Date
             └── Shipped Date
```

---

## Relationships

The fundamental relationship pattern in a dimensional model is **dimension-to-fact, one-to-many**.

```text
Dimension
    1
    │
    │
    N
 Fact Table
```

For this model:

| Dimension | Relationship | Fact |
|---|---|---|
| `Dim_Customer` | 1:N | `Fact_Orders` |
| `Dim_Employee` | 1:N | `Fact_Orders` |
| `Dim_Product` | 1:N | `Fact_Orders` |
| `Dim_Category` | 1:N | `Fact_Orders` |
| `Dim_Date` | 1:N | `Fact_Orders` |
| `Dim_Territory` | 1:N | `Fact_Orders` |

The fact table is therefore at the **many side** of almost every analytical relationship.

---

## Example Analytical Query Path

Suppose the business asks:

> **What was total revenue by product category and territory during each quarter?**

The query conceptually follows:

```text
Dim_Category
      │
      │
      ▼
 Fact_Orders ◄──── Dim_Territory
      ▲
      │
      │
   Dim_Date
```

The dimensions provide the descriptive attributes:

```text
Category  → category_name
Territory → territory_description
Date      → year, quarter
```

The fact table provides the measure:

```text
SUM(extended_amount)
```

The resulting report might look like:

```text
Year | Quarter | Category  | Territory | Revenue
-----|---------|-----------|-----------|---------
2026 | Q1      | Beverages | Southwest | $125,400
2026 | Q1      | Produce   | Southwest | $ 84,200
2026 | Q1      | Beverages | Northeast | $142,750
```

This separation between **descriptive dimensions** and **numeric facts** is the central principle of dimensional modeling.

---

## Relational vs. Dimensional Model

The source relational model is optimized around operational entities:

```text
Customer
    │
    └── Order
          │
          └── OrderDetail
                 │
                 └── Product
                         │
                         └── Category
```

The dimensional model reorganizes those relationships around the business event:

```text
                    Dim_Date
                       │
                       │
Dim_Customer ──── Fact_Orders ──── Dim_Employee
                    /  |  \
                   /   |   \
          Dim_Product  |   Dim_Territory
                       |
                  Dim_Category
```

The difference reflects two different design objectives.

The **relational model** is primarily concerned with:

- Transaction processing
- Data integrity
- Normalization
- Reducing redundancy
- Efficient inserts and updates

The **dimensional model** is primarily concerned with:

- Analytics
- Aggregation
- Reporting
- Understandable business structures
- Efficient analytical queries

---

## Summary

The Northwind dimensional model transforms operational order data into an analytical structure centered around `Fact_Orders`.

The grain is defined as:

> **One fact row per product line item on an order.**

The dimensions provide analytical context:

```text
Customer  → WHO purchased?
Employee  → WHO handled the sale?
Product   → WHAT was sold?
Category  → WHAT type of product?
Territory → WHERE was it sold?
Date      → WHEN was it sold?
```

The fact table provides the measurable business event:

```text
Fact_Orders
    │
    ├── quantity
    ├── unit_price
    ├── discount
    └── extended_amount
```

Together, these structures form a dimensional model suitable for **data warehousing, BI reporting, dashboards, KPI calculation, trend analysis, and downstream analytics**.

The most important design decision is the **grain of the fact table**. Once the grain is established as one row per order line item, the dimensions and measures must consistently describe that same level of detail.
````
