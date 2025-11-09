## Day 2: Core Azure Services Setup & Rationale

To build this pipeline, we must first provision our core cloud infrastructure. This document explains the setup and, more importantly, the **rationale** for choosing each service.

---

### 1. Azure Data Lake Storage (ADLS) Gen2

This is the foundation for our batch processing (Pipeline 1).

**Why ADLS Gen2 instead of standard Blob Storage?**
* **Blob Storage** is a simple "object store." It's like a digital drawer where you throw files.
* **ADLS Gen2** is built *on* Blob Storage but adds a **Hierarchical Namespace (HNS)**. This is the key difference. HNS allows for *true folders* and directories, just like on your local PC.
* **For analytics, HNS is critical.** It makes operations like moving, renaming, and listing files in massive datasets *infinitely faster* than in Blob Storage. This is essential for any serious data pipeline.

**Our Container Strategy (Batch Loading)**
We created two containers:
1.  **`Raw data container` (The "Source"):** This is the landing zone. New customer batch files arrive here. Our ADF pipeline *only reads* from this location.
2.  **`Archive data container` (The "Audit Trail"):** After a file is successfully processed by `Pipeline 1` (copied to Synapse), it is *moved* from `Raw` to `Archive`.

**Why two containers?**
This is a standard, resilient pipeline pattern. It prevents processing the same file twice and gives us a perfect audit trail. If our Synapse table ever gets corrupted, we can easily re-load the data from the `Archive` container.

**Setup Steps:**
1.  Create a new `Storage Account`.
2.  In the `Advanced` tab during setup, **enable `Hierarchical namespace`**. This is the step that turns it into an ADLS Gen2.
3.  Inside the account, create two `Containers`: `raw-data` and `archive-data`.

---

### 2. Azure Cosmos DB

This is the engine for our real-time processing (Pipeline 2).

**Why Cosmos DB?**
For our `Booking` data, we need to capture changes *as they happen*. We can't wait for a nightly batch. Cosmos DB is a high-speed NoSQL database with one killer feature for this project: the **Change Feed**.

The **Change Feed** is a persistent log of all changes (inserts and updates) to our bookings container. Our `ADF Data Flow` (Pipeline 2) can subscribe directly to this feed, allowing us to process booking data **incrementally** in near-real-time. This is the foundation of our CDC (Change Data Capture) strategy.

**Setup Steps:**
1.  Create an `Azure Cosmos DB account` (using the Core (SQL) API).
2.  Create a new `Database`.
3.  Create a new `Container` named `bookings` to hold our booking data. The Change Feed is enabled by default.

---

### 3. Azure Synapse Analytics (Dedicated SQL Pool)

This is our central Data Warehouse (DWH) and the "single source of truth."

**Top-Notch Rationale: Why Dedicated SQL Pool vs. Serverless?**

This was a critical architectural decision. The choice comes down to **Storage vs. Query Engine** and **Performance vs. Exploration**.

* A **Serverless Pool** is a *query-per-pay* engine. It is not a database. It simply allows you to run SQL queries on top of files *already in the data lake* (e.g., Parquet, CSV). It's built for **ad-hoc data exploration**, data science, and situations where you want to query data "where it lives." It has a variable "cold start" performance and is not optimized for high-concurrency BI.

* A **Dedicated SQL Pool** is a true, provisioned **Massively Parallel Processing (MPP)** data warehouse. It *stores* data in its own highly-optimized, columnar format. This is the key. By provisioning compute (e.g., `DW100c`), we are buying **guaranteed, consistent performance**.

**For this project, the Dedicated Pool was the only correct choice for three reasons:**

1.  **Production-Ready Performance:** Our goal is to power *downstream applications* (the n8n AI agent and future Power BI dashboards). These applications demand fast, predictable, sub-second query responses. A Serverless pool simply cannot guarantee this.
2.  **High Concurrency:** A Serverless pool struggles with many users at once. A Dedicated pool is *built* for high-concurrency BI, where dozens of users or applications are querying the warehouse simultaneously.
3.  **The "Single Source of Truth":** Our pipeline's job is to *create* a final, clean, structured, and reliable data asset. A Serverless pool just *queries* the lake; a **Dedicated pool *is* the final asset**. It's the persistent "single source of truth" for the entire business.

**In short: We chose Serverless for *exploration*, but Dedicated for *production*.**

**Setup Steps:**
1.  Create an `Azure Synapse Workspace`.
2.  Inside the workspace, navigate to the `Manage` hub.
3.  Select `SQL pools` and create a new `Dedicated SQL pool`.
4.  Set the performance level (e.g., `DW100c` is a good starting point for a project).