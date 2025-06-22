# CMA Architecture: Stage-wise Details

## 1. Data Sources (FISS, GOS, IMATCH)
**Responsibility:**
- Provide daily trade and settlement data from external systems.

**What it does:**
- Sends raw data files or events (e.g., trades, settlements, reconciliations) to the CMA system for processing.

---

## 2. Ingestion Layer (Java/Spring Boot)
**Responsibility:**
- Receive, parse, and standardize incoming data.

**What it does:**
- Listens for new files or events from FISS, GOS, IMATCH.
- Parses and validates the format.
- Tags each event with an `entity_id` (to identify the fund, legal entity, or account group).
- Publishes standardized events to the Event Bus (Kafka).

---

## 3. Event Bus (Kafka, partitioned by entity_id)
**Responsibility:**
- Decouple services and enable parallel, entity-wise processing.

**What it does:**
- Receives events from the Ingestion Layer.
- Partitions events by `entity_id` so each entity's data is processed independently.
- Buffers and delivers events to downstream services (Orchestration Engine, Entity-Wise Processor).

---

## 4. Orchestration Engine (Java/Spring Boot)
**Responsibility:**
- Manage and trigger entity-specific workflows.

**What it does:**
- Listens to Kafka for new events.
- Determines which workflow(s) to trigger for each entity.
- Coordinates the sequence of processing steps for each entity.
- Sends workflow triggers to the appropriate Entity-Wise Processor.

---

## 5. Entity-Wise Processor (Java/Spring Boot, parallel per entity)
**Responsibility:**
- Execute business logic for each entity independently.

**What it does:**
- Processes events for its assigned entity (fund, legal entity, or account group).
- Handles calculations, transformations, and business rules.
- Passes processed data to Validation/Enrichment services.

---

## 6. Validation/Enrichment Services (Java/Spring Boot)
**Responsibility:**
- Validate data against CASS rules and enrich with reference data.

**What it does:**
- Checks data for completeness, accuracy, and compliance with FCA CASS rules.
- Enriches data using reference data (e.g., security master, account mappings).
- Flags or rejects non-compliant or incomplete data.
- Passes validated/enriched data to Asset Classification.

---

## 7. Asset Classification & Lockup Logic (Java/Spring Boot)
**Responsibility:**
- Classify assets and apply lockup logic as per CASS requirements.

**What it does:**
- Determines asset types (e.g., client money, firm money, safe custody assets).
- Applies lockup and segregation logic as required by CASS.
- Ensures assets are correctly classified and locked up.
- Passes results to Audit/Compliance Tracker and Database.

---

## 8. Audit/Compliance Tracker (Java/Spring Boot)
**Responsibility:**
- Maintain a complete audit trail and generate compliance alerts.

**What it does:**
- Logs all actions, decisions, and data changes for each entity.
- Generates alerts for compliance breaches or exceptions.
- Stores audit logs in the Database for reporting and regulatory review.

---

## 9. Database (MS SQL, entity-tagged)
**Responsibility:**
- Persist all processed, validated, and classified data.

**What it does:**
- Stores all data, tagged by `entity_id` for easy retrieval.
- Supports efficient queries for entity-level, CASS-compliant reporting.
- Retains audit logs, validation results, and classification outcomes.

---

## 10. UI Dashboard (React)
**Responsibility:**
- Provide real-time, entity-level visibility and reporting.

**What it does:**
- Queries the Database for the latest data, status, and alerts.
- Displays dashboards, reports, and compliance status for each entity.
- Enables users to drill down into audit logs, validation results, and exceptions.

---

## Summary Table

| Stage                        | Responsibility                                      | What it Does                                                      |
|------------------------------|-----------------------------------------------------|-------------------------------------------------------------------|
| Data Sources                 | Provide trade/settlement data                       | Send raw data to CMA                                              |
| Ingestion Layer              | Parse and standardize data                          | Parse, tag, and publish events to Kafka                           |
| Event Bus (Kafka)            | Decouple and parallelize processing                 | Partition and deliver events by entity                            |
| Orchestration Engine         | Manage workflows                                    | Trigger entity-specific workflows                                 |
| Entity-Wise Processor        | Execute business logic per entity                   | Process, transform, and pass data to validation                   |
| Validation/Enrichment        | Validate and enrich data                            | CASS checks, enrich, flag issues                                  |
| Asset Classification/Lockup  | Classify and lock up assets                         | Apply CASS logic, pass to audit and DB                            |
| Audit/Compliance Tracker     | Log and alert for compliance                        | Store audit trail, generate alerts                                |
| Database (MS SQL)            | Persist all data                                    | Store all processed, validated, and classified data               |
| UI Dashboard (React)         | Real-time reporting                                 | Display dashboards, reports, and compliance status                | 