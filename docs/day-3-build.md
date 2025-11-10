## Day 3: Building the Data Warehouse

On Day 2, we provisioned our empty Azure services. Today, we build the internal structure of our Data Warehouse and create the "digital plumbing" (Linked Services) in Azure Data Factory (ADF) to connect all our services.

---

### 1. Building the Synapse DWH Structure

We executed our `synapsetablecreation.sql` script (found in the `sql_queries` folder) in our Synapse Dedicated SQL Pool. This script performs three critical tasks:

#### a. Creates the Schema
`CREATE SCHEMA airbnb;`
This simply creates an organized, dedicated workspace for our project tables inside the warehouse.

#### b. Creates the Core Tables (Star Schema)
This script builds a classic "Star Schema" model, which is the industry standard for analytics.

* **`airbnb.dim_customer` (Dimension Table):**
    This table holds the "who." It stores descriptive, non-changing (or slowly-changing) information about our customers (name, email, address, etc.). `Pipeline 1` will be responsible for loading this.

* **`airbnb.fact_booking` (Fact Table):**
    This table holds the "what." It stores the individual transaction *events*—in this case, each booking. It contains "facts" (like `total_amount`, `nights`) and "foreign keys" (`customer_id`, `listing_id`) that link back to our dimension tables. `Pipeline 2` (our CDC pipeline) will be responsible for loading this.

* **Idempotent Design:** Notice the `IF OBJECT_ID... DROP TABLE` command. This is critical. It makes our script **idempotent**, meaning we can re-run it many times without it failing. It simply drops the old table and creates a fresh one.

---
### 2. The Performance Layer: Why We Use Aggregations

This is the most critical design decision in our warehouse. We *could* just stop at the fact table, but we'd have a slow, expensive system.

**The "Why": From a Slow-Scan to a Sub-Second Query**

**The Problem:** Without an aggregation, any report (like Power BI) or app (our n8n workflow) needing "total bookings by country" would have to scan the *entire* `fact_booking` table—potentially millions or billions of rows—and perform a complex `JOIN` and `GROUP BY`... **every single time** the page is refreshed. This is slow, expensive, and scales poorly.

**The Solution:** We create a "BI Acceleration" layer. We perform the expensive query *once* during our data load and store the small, clean results.

#### a. `airbnb.BookingCustomerAggregation` (The "BI Accelerator" Table)

This is our pre-calculated summary table.

* **What it is:** A small, wide table that stores the *answers* to our most common business questions (e.g., `total_bookings`, `cancellation_rate` by `country`).
* **The Benefit:** Our downstream tools (n8n, Power BI) query *this* tiny table (maybe 100 rows) instead of the 10-million-row `fact_booking` table.
* **The Result:** This is the difference between a **10-second** dashboard refresh (unacceptable) and a **100-millisecond** one (instant). It drastically reduces the compute load on our warehouse and ensures a fast, reliable user experience.

#### b. `airbnb.BookingAggregation` (The "ELT Engine" Stored Procedure)

This is the *engine* that builds our accelerator table.

* **Why a Stored Procedure? Performance (ELT vs. ETL):**
    This is the core "top-notch" reason. We *could* use an ADF Data Flow to read from the fact table, aggregate it, and write it back. This is an **ETL** (Extract-Transform-Load) approach. It's **slow** because the data must *leave* the Synapse database, travel over the network to the ADF integration runtime, get processed, and then travel *back* to Synapse.

    Instead, we use a Stored Procedure. This is an **ELT** (Extract-Load-Transform) approach. The `TRUNCATE` and `INSERT...SELECT...GROUP BY` logic runs **inside the Synapse MPP engine**. The data never leaves the database. This is "pushdown" logic and is **orders of magnitude faster** than the ETL method.

* **Why a Stored Procedure? Encapsulation & Reusability:**
    The complex aggregation logic is **encapsulated** (contained) in a single, version-controlled `.sql` script. It's reusable and clean. Our ADF pipeline's job becomes simple: at the end of the data load, it just has to make one call: `EXEC airbnb.BookingAggregation`. This is clean, maintanable, and a best-practice for data warehousing.

---

