Project Architecture Breakdown
This illustrates the high-level architecture of the end-to-end AirBnB data pipeline, from raw data ingestion to intelligent automation.

Here is the flow:
![Architectural diagram](./architecture-diagram.png)

1.Dual-Source Ingestion:
-Azure Cosmos DB (CDC): The pipeline captures every booking change in near-real-time by tapping directly into the Cosmos DB Change Feed (CDC). This event-driven approach is for our high-velocity, transactional data.
-Azure Data Lake Storage (ADLS): This stores our batch data. Raw customer files are landed in the Raw data container for processing.

2.Orchestration & Transformation (The "Brain"):
-Azure Data Factory (ADF): This is the central hub that orchestrates the entire workflow. It contains three key pipelines:

PIPELINE 1 (LOAD_CUSTOMERS_DIM): Ingests the batch files from ADLS to build and update the CUSTOMER_TABLE.

PIPELINE 2 (LOAD_BOOKINGS_FACT): Ingests the real-time CDC data from Cosmos DB to build the BOOKING_TABLE.

PIPELINE 3 (FINAL_PIPELINE): The master pipeline that manages the execution and dependencies of the other two, ensuring data integrity.

Warehousing (The "Truth"):

Azure Synapse Analytics (DWH): This is our single source of truth. ADF loads the cleaned and transformed data into Synapse, populating the dimensional (CUSTOMER_TABLE) and fact (BOOKING_TABLE) tables. It also creates a BOOKING_AGGREGATE table for pre-calculated, high-speed analytics.

Intelligent Action (The "Value"):

n8n AI Agentic Workflow: This is the final, value-driving step. The n8n automation platform queries the clean, aggregated data from Synapse. It then feeds these insights to an AI workflow to trigger automated, intelligent actions (like generating reports or sending personalized alerts).
