# AirBnB CDC & AI-Powered Ingestion Pipeline

This is an end-to-end, event-driven data platform on Azure that turns real-time data into AI-driven actions.

> 🚧 **Project Status: Work in Progress** 🚧
> I am actively building this project. This repository will be updated day-by-day to showcase the development process.

---

## 🗺️ Day 1: The Architectural Blueprint

As the first step, I have designed the complete high-level architecture for the data pipeline, from ingestion to intelligent action.

![Project Architecture](./docs/architecture-diagram.png)


---

## Project Goal
This project demonstrates a complete, event-driven loop: it ingests customer booking data in real-time, feeds it to an AI agent, and automatically sends a personalized booking status acknowledgement right back to the customer.

---

### Day 2: Cloud Infrastructure Setup
Detailed the setup and rationale for our core services: ADLS Gen2, Cosmos DB, and Synapse Analytics.
* **[Click here to see the Service Setup & Rationale](./docs/day-2-setup.md)**

### Day 3: DWH Build & ADF Connections
Created the star-schema tables, high-performance aggregation layer, and all ADF Linked Services.
* **[Click here to see the DWH scripts](./docs/day-3-build.md)**

### Day 4: Mock Data Generation (TDD)
Documented the Test-Driven Development (TDD) approach, explaining the "why" and "how" of generating mock data for both batch (SCD) and real-time (CDC) pipelines.
* **[Click here to see the Mock Data Rationale](./docs/day-4-mock-data.md)**

### Day 5: Synapse DWH Build & SQL Rationale
Committed the foundational SQL script for building the DWH and documented the "why" behind its idempotent and performance-optimized (ELT) design.
* **[Click here to see the Synapse SQL Rationale](./docs/day-5-synapse-build.md)**
* **[Click here to see the SQL Build Script](./sql_queries/synapse_table_creation.sql)**