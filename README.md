# prompts

You are a senior banking solutions architect.

Design a production-ready loan product ingestion system using ReactJS (frontend), Node.js with Express (backend), and SQL Server (database).

The system ingests monthly Excel uploads (~2500 loan products) for a banking loan department. Each loan product includes product ID, product name, start date, withdrawn date, and pricing. Pricing may change multiple times per year.

Key requirements:

Single-writer system (no concurrent updates from multiple systems)

Use a staging + batch-driven architecture for auditability and regulatory compliance

Track upload batches with full lineage (who, when, status, counts)

Validate data before committing to production tables

Use TVP-based MERGE for optimized bulk insert/update into core tables

Maintain full history of changes (pricing changes over time, with timestamps and batch IDs)

Allow replay and recovery of failed batches

Provide end-user feedback after upload showing:

total records processed

how many were newly created

how many were updated

list of created and updated product names

Deliver:

High-Level Design (HLD)

Low-Level Design (LLD)

Database schema (staging, core, history, batch tables)

SQL Server MERGE with OUTPUT examples

Node.js API flow and response mapping

UI behavior for displaying pricing change history

Non-functional requirements (performance, audit, security, recovery)

The design should follow bank-grade best practices and be suitable for production deployment.

====================================================================================================

Short Prompt (When You Want Fast but Correct Output)

Design a banking-grade batch ingestion system for loan products using ReactJS, Node.js (Express), and SQL Server.

Monthly Excel uploads (~2500 records) must be staged, validated, batch-tracked, merged into core tables using TVP + MERGE, and fully audited with change history.

Include HLD, LLD, schema, MERGE OUTPUT, Node.js response mapping, retry/replay strategy, and UI behavior for pricing changes.


==========================================================================================================

Architecture-Only Prompt (For HLD Reviews / Interviews)

As a banking architect, explain a staging-driven, batch-controlled data ingestion architecture for loan products with monthly Excel uploads.

Cover auditability, recovery, TVP-based MERGE optimization, history tracking, and user feedback.

Assume a single-writer system and strict regulatory requirements.
