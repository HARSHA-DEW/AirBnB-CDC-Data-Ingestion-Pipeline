## Day 4: Mock Data Generation (A Test-Driven Approach)

Today's work is not about the pipeline itself, but the critical, professional practice of **Test-Driven Development (TDD)**. Before we build our pipelines, we must first build a reliable way to *test* them.

### The "Smart Rationale": Why Mock Data is Better than a Live API

A junior developer might try to connect to a live, 3rd-party AirBnB API. This is a critical mistake during development for several reasons:

1.  **Reliability & Stability:** Live APIs go down. They have rate limiting. You can get your IP banned. You cannot build a stable pipeline if your source is unstable.
2.  **Cost:** Most high-quality, real-time APIs are not free.
3.  **The "Happy Path" Problem:** A live API gives you *real* data. It *won't* give you the weird, broken, or challenging data you need to prove your pipeline is robust.
4.  **Engineering Edge Cases:** Our mock data approach allows us to *deliberately* engineer the difficult scenarios. What if a booking has a `null` price? What if a customer's `country` is misspelled? Our pipeline *must* handle this. We can only test this by *creating* that bad data.

Using mock data is a deliberate choice. It isolates our system, allowing us to prove that *our* logic works and that our pipeline is resilient to the "unhappy paths" and dirty data that define all real-world data engineering.

---

### 1. Simulating Batch Data (Pipeline 1): The ADLS Files

To test our `Pipeline 1` (the batch customer load), we created a set of CSV files.

* **`customer_data_2025_10_30_base.csv`**: This is our **Base Load**. It represents the initial, large set of customer data we get on Day 1.
* **`...delta1.csv` & `...delta2.csv`**: These are our **Delta (Incremental) Loads**. They represent the *new* files that arrive each day.
    * These files contain a mix of **new customers** (for testing `INSERT` logic) and **updated information** for existing customers (for testing our `SCD Type-1` / `UPDATE` logic).

**Generation Method:**
These files were generated using a Python script with the **`Faker`** library. `Faker` allows us to create millions of rows of realistic, yet entirely fake, data (names, addresses, emails) that follows a consistent schema.

### 2. Simulating Real-Time Events (Pipeline 2): The Cosmos DB Script

To test our `Pipeline 2` (the real-time CDC), we can't use static files. We need to *simulate events*.

* **`mock_data_in_cosmosdb.py`**: This is our "Event Generator" script.
* **How it Works:**
    1.  It uses the **`Faker`** library to create a realistic, new JSON object for a single booking.
    2.  It uses the **`azure-cosmos`** Python SDK to connect to our Cosmos DB instance.
    3.  It **inserts** this new JSON document into the `bookings` container.
* **The Result:** The instant this script runs, Cosmos DB sees a new `INSERT`. This immediately fires the **Change Feed**, which is what our `Pipeline 2` is subscribed to. This script effectively *is* the customer making a new booking, allowing us to test our CDC pipeline from end-to-end.