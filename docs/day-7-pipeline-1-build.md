## Day 7: Building Pipeline 1 - Batch Customer Dimension Load

Today, we are building our first Azure Data Factory (ADF) pipeline: `Pipeline 1 - Load Customers Dimension Table`. This pipeline is responsible for ingesting new customer files from our ADLS Gen2 `raw-data` container into the `airbnb.dim_customer` table in Synapse.

This pipeline is designed for **resilience, idempotency, and scalability** for batch processing.

---

### Pipeline 1 Architecture Diagram

This diagram, which we discussed on Day 1, shows the core flow of our batch customer data pipeline.

![Pipeline 1 Architecture Diagram](./pipeline1.png)



*(Note: Ensure you have placed the image file `pipeline-1-diagram.png` into your `docs` folder or adjusted the path if it's elsewhere.)*

---

### Step-by-Step ADF Build & Rationale

We navigate to Azure Data Factory Studio and begin building our pipeline:

#### 1. Create a New Pipeline
* In ADF Studio, go to the `Author` tab.
* Click the `+` sign and select `Pipeline`. Name it `Pipeline_1_Load_Customer_Dimension`.

#### 2. Get Metadata Activity
* **Drag & Drop:** From the `Activities` pane, drag a `Get Metadata` activity onto the canvas.
* **Settings:**
    * **Dataset:** Create a new `DelimitedText` (CSV) Dataset.
        * Point it to our `adlsconn` Linked Service.
        * Set the **File path** to the `raw-data` container in ADLS Gen2. Crucially, **do NOT specify a file name or folder path here**, as we want it to list *all* files.
    * **Field List:** Add `Child Items` to the list of arguments. This tells the activity to return a list of all files/folders in the specified path.
* **Rationale:** This makes our pipeline **dynamic and metadata-driven**. Instead of hard-coding file names, it automatically discovers all new files in the `raw-data` container that need processing. This is robust and requires no code changes when new files arrive.

#### 3. For Each Loop Activity
* **Drag & Drop:** Drag a `For Each` activity onto the canvas.
* **Connect:** Connect the `Get Metadata` activity's success output to the `For Each` activity.
* **Settings:**
    * **Items:** Set this to `@activity('Get Metadata1').output.childItems`. This passes the list of discovered files from the `Get Metadata` activity to the loop.
    * **Batching:** Set `Sequential` to `True`.
* **Rationale:** The `For Each` loop is crucial for **resilience and error handling**. By setting `Sequential` to `True` and processing **one file at a time**, if one customer file has bad data and fails, it only stops *that specific file's processing*, not the entire batch. Other files can still be processed successfully. This isolates failures.

#### 4. Inside the For Each Loop: The Core Activities

Now, open the `For Each` loop and add these three activities in sequence:

##### a. `Copy Data` Activity (Copy Customers Data to Synapse)
* **Drag & Drop:** Drag a `Copy Data` activity inside the loop.
* **Settings:**
    * **Source:**
        * **Source Dataset:** Create a new `DelimitedText` (CSV) Dataset.
        * Point it to `adlsconn`.
        * **File Path:** Crucially, set this dynamically using an expression: `@item().name`. This means each iteration of the loop processes the *current* file's name.
        * Ensure `First row as header` is checked.
    * **Sink (Destination):**
        * **Sink Dataset:** Create a new `Azure Synapse Analytics` Dataset.
        * Point it to our `synapseconn` Linked Service.
        * Select the `airbnb.dim_customer` table.
        * **Table Option:** Choose `Auto create table` (for initial setup) or `None` if the table is already created on Day 5.
        * **Pre-copy script:** This is where we implement **SCD Type-1 logic (Upsert)**. Use a stored procedure or an `MERGE` statement here to handle updates for existing customers. *(Example: `MERGE INTO airbnb.dim_customer AS TGT USING #StagingTable AS SRC ON TGT.customer_id = SRC.customer_id WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT ...;`)*
* **Rationale:** This is the "L" (Load) part of our ELT. Data is moved from the raw zone to our dimension table. The dynamic file path makes it flexible, and the pre-copy script enables **SCD Type-1**, ensuring our customer dimension data is always up-to-date.

##### b. `Copy Data` Activity (Copy to Archive)
* **Drag & Drop:** Drag another `Copy Data` activity inside the loop, connecting it to the success of the previous `Copy Data` activity.
* **Settings:**
    * **Source:** Re-use the same `DelimitedText` (CSV) Dataset used for the Synapse copy, dynamically set to `@item().name`.
    * **Sink (Destination):**
        * **Sink Dataset:** Create a new `DelimitedText` (CSV) Dataset.
        * Point it to `adlsconn`.
        * **File Path:** Set the container to `archive-data` and the filename to `@item().name`.
        * **Copy behavior:** `None` (just copy the file).
* **Rationale:** This provides an **immutable audit trail**. After a file is successfully processed into Synapse, a copy of the *original raw file* is placed into the `archive-data` container. This is crucial for data governance, disaster recovery, and debugging, providing an exact historical record of all ingested raw data.

##### c. `Delete` Activity (Delete Source File)
* **Drag & Drop:** Drag a `Delete` activity inside the loop, connecting it to the success of the `Copy to Archive` activity.
* **Settings:**
    * **Source Dataset:** Use the same `DelimitedText` (CSV) Dataset, dynamically set to `@item().name`.
* **Rationale:** This ensures **idempotency and prevents reprocessing**. Only after a file has been successfully loaded into Synapse *and* securely archived is it deleted from the `raw-data` container. This guarantees that the next pipeline run will not re-process the same file, preventing duplicate data and keeping our raw zone clean.

---

### Final Steps: Publish & Test
* Once all activities are configured, click `Publish All` to save your pipeline.
* **Testing:** Upload a mock customer CSV file to your `raw-data` container and trigger the pipeline. Verify data in Synapse and check the `archive-data` container.