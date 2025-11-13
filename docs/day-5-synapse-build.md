## Day 5: Synapse DWH Build & SQL Rationale

Today, we are committing the foundational SQL script (`synapse_table_creation.sql`) that builds our entire Data Warehouse structure inside the Synapse Dedicated SQL Pool.

This documentation file explains the *design and rationale* behind that script, which is located in the `sql_queries/` folder. The script is broken into 5 sections.

---

### Line-by-Line "Top-Notch" Rationale

#### 1. `CREATE SCHEMA airbnb`
* **What it does:** Creates a "folder" or "namespace" called `airbnb`.
* **Why it matters:** This is a basic data governance best-practice. We don't dump our tables into the default `dbo` schema. This keeps our project's tables organized, separate from other projects, and makes security management much easier.

#### 2. `IF OBJECT_ID... DROP TABLE` (Idempotency)
* **What it does:** Before creating a table, the script checks if a table with that name *already exists*. If it does, it drops (deletes) it first.
* **Why it matters:** This is **critical**. It makes our script **idempotent**, which means we can run it 1,000 times and it will produce the same, correct result without failing. Our ADF pipeline can now run this script at the start of a build, guaranteeing a clean slate every time.

#### 3. `dim_customer` & `fact_booking` (The Star Schema)
* **What they do:** The script creates our core Star Schema: a `dim_customer` table (for "who") and a `fact_booking` table (for "what").
* **Why it matters:** By separating descriptive data (Dimensions) from event data (Facts), we make our warehouse incredibly fast to query. We avoid storing the customer's name and country on *every single booking row*, which saves massive amounts of space and speeds up queries.

#### 4. `BookingCustomerAggregation` (The BI Accelerator)
* **What it does:** The script creates a small, pre-calculated summary table.
* **Why it matters (Performance):** This is our "BI Acceleration" layer. Without this, a Power BI report needing "total revenue by country" would have to scan the 10-million-row `fact_booking` table *every time*. That's a slow, 10-second dashboard. By using this table, the BI tool queries a 100-row aggregation table. The result is a **100-millisecond** dashboard. We do the heavy lifting *once* so our users can have instant analytics.

#### 5. `BookingAggregation` (The ELT Engine Stored Procedure)
* **What it does:** The script creates a reusable command (`EXEC airbnb.BookingAggregation`) that truncates and re-loads our aggregation table.
* **Why it matters (ELT vs. ETL):** This is the heart of our performance strategy.
    * **ETL (Bad):** We could use an ADF Data Flow to pull 10 million rows from Synapse, aggregate them, and write the 100 rows back. This is **slow** as data travels over the network.
    * **ELT (Good):** We use this Stored Procedure. Our ADF pipeline just says `EXEC...`. The **Synapse MPP engine** does all the work *inside* the database (this is "pushdown" logic). The data never leaves. This is **orders of magnitude faster** and the standard for modern data warehousing.