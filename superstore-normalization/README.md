# 🏬 Superstore Sales Dataset Normalization

## 📌 Project Overview
The goal of this project was to convert a single, flat **Superstore Sales CSV dataset** (9,994 transactional records) into a fully normalized **Star Schema** in MySQL. 

By separating flat transactional logs into structured dimension and fact tables, this implementation eliminates data redundancy, enforces relational integrity, and prepares the data for efficient analytical querying and reporting.

---

## 🏗 Data Model & Schema Architecture

The database was redesigned into 1 central **Fact Table** surrounded by 5 **Dimension Tables**:

### **Dimension Tables**
- **`dim_customer`**: Stores unique customer details (`Customer_id`, `Customer_name`, `Segment`).
- **`dim_address`**: Captures geographic details (`Address_id`, `Country`, `Region`, `State`, `City`, `Postal_code`).
- **`dim_product_category`**: High-level classification hierarchy (`Category_id`, `Category`, `Sub_category`).
- **`dim_product`**: Product catalog referencing categories (`Product_id`, `Product_name`, `Category_id`).
- **`dim_date`**: Time-intelligence table populated dynamically via a MySQL Stored Procedure for dates between `2014-01-01` and `2018-12-31`.

### **Fact Table**
- **`fact_orders`**: The core transactional table containing order line items, financial measures (`Sales`, `Quantity`, `Discount`, `Profit`), and surrogate/foreign keys linking to all dimension tables (`Order_id`, `Customer_id`, `Address_id`, `Product_sk`, `Order_date_key`, `Ship_date_key`).

---

## 🛠 Key Implementation Steps

1. **Staging & Cleaning**: Imported raw transactional data into a staging table (`raw_superstore`).
2. **Dimension Building**: Extracted distinct entities to create and populate dimension tables.
3. **Automated Date Key Generation**: Wrote a custom MySQL Stored Procedure (`sp_populate_dim_date`) to dynamically build date dimension keys for order/ship date tracking.
4. **Fact Table Construction**: Built `fact_orders` by joining raw records with dimension tables using surrogate keys to maintain data integrity.

---

## 📄 Included Files

- **`normalization.md`**: Step-by-step SQL scripts, table definitions, and DDL/DML queries used during implementation.
- **`diagram.png`**: Interactive Entity-Relationship Diagram (ERD) generated from MySQL showing table relationships and foreign keys.

---

## 💻 Tech Stack
- **Database Engine**: MySQL
- **Tooling**: VS Code (DBCode Extension), Markdown
