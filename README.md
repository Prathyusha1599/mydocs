Purview – UKG Data Governance Summary
1. Introduction
Microsoft Purview is being implemented to provide data governance for UKG employee data stored in Fabric Lakehouse. The goal is to ensure sensitive information (e.g., salaries, employee IDs, employment dates) is protected, while still allowing business users to analyze and report on the data.

The Purview–Fabric integration has been completed and sensitivity labels are now propagating to Power BI. This document summarizes Purview capabilities relevant to UKG data, identifies limitations, and recommends short- and long-term approaches.
2. Current Implementation Status
•	✅ Purview–Fabric integration fully configured.
•	✅ Sensitivity labels applied at the Lakehouse level.
•	✅ Labels propagate correctly into Power BI.
•	✅ Row-Level Security (RLS) is available for restricting which rows of data (e.g., employees from Rhode Island vs. other locations) a user can see.
3. Key Limitations Identified
3.1 Sensitivity Labels
- Labels apply at the entire Lakehouse level, not at the table or column level.
- Cannot assign different labels to employee data vs. hours worked data.
- Cannot hide specific sensitive columns (like salary or employee ID).
3.2 Row-Level Security (RLS)
- RLS can restrict which employees (rows) a user sees.
- Cannot hide columns → if someone can view an employee record, they see all associated fields.
3.3 Column-Level Security (CLS)
- Not supported natively in Purview or Fabric.
- Only achievable via Power BI Object-Level Security (OLS) using external tools (e.g., Tabular Editor).
- Challenges with OLS:
   * Not supported in Power BI Desktop.
   * Can force models into DirectQuery mode (performance impact).
   * Not part of original SOW and requires separate effort.
4. Implications for UKG Data
- Business cannot currently achieve fine-grained control (e.g., hiding salary while showing hours).
- RLS solves location-based filtering (RI employees see RI data).
- Column hiding requires separate design choices (outside of Purview’s scope).
5. Recommendations
Short-Term (Quick Wins)
- Use Row-Level Security (RLS) to restrict data visibility by plant/location.
- Apply Sensitivity Labels at dataset level to classify UKG data for compliance.
- Clearly document limitations to set stakeholder expectations.
Long-Term (Future Options)
1. Option 1: Separate Datasets / Lakehouses
   - Create one dataset/lakehouse/report for sensitive data (salary, IDs).
   - Create another for non-sensitive data (general reporting).
   - Grants business control without needing unsupported features.

2. Option 2: Power BI Object-Level Security (OLS)
   - Use OLS to hide columns like salary.
   - Be aware of performance trade-offs and added complexity.
   - Requires additional setup and not part of current contract.

3. Option 3: Alternative Reporting Solution
   - Evaluate other reporting tools that natively support column-level security if this becomes a mandatory compliance requirement.
6. Conclusion
Purview provides strong integration with Fabric for discovery, sensitivity labeling, and lineage. However, it currently lacks column-level security controls. For UKG data governance:
- RLS + Sensitivity Labels cover immediate needs (row filtering + classification).
- Column-level restrictions (e.g., hiding salary fields) require either dataset separation or Power BI OLS as a separate project.

Next Steps:
- Deliver this summary to stakeholders.
- Decide whether to proceed with short-term RLS-only approach or invest in long-term OLS/separate dataset design.

------------------------------------------------------------

InforM3 Cloud → Local → Lakehouse Auditing & Repair — Discovery Summary
Executive summary

We ingest M3 data to the lakehouse via Export MI (ION) API; the local SQL database is refreshed in batches (≈ every 10 minutes), so cloud vs. local are rarely in perfect sync.

A daily audit compares prior-day inserts (today excluded) between cloud and local; it flags positives (missing inserts locally) but intentionally ignores updates/deletes.

Recurring mismatches concentrate in a few tables—especially MSYTXL (text lines) and MSYTXH (text headers)—while most other tables reconcile.

For deeper validation (and to catch deletes), a deep-dive audit rolls back day-by-day to find exact bad dates and supports targeted repairs.

Not all tables support the same strategy:

With LMTS (timestamp): full incremental audit/repair possible.

No LMTS but has created date: limited audit; repairs by date windows.

No dates at all (e.g., caches): only full reload (“Hulk-Smash”) works.

Evidence from today’s materials

Daily audit email (2025-09-10 08:00) shows non-zero deltas: MSYTXL +7, MSYTXH +2, plus a few smaller ones (e.g., MPAGRI +3).

SQL audit screen sets pStartDate=20240901; results show mostly LMTSDiff=0 with occasional small ±1 drifts, reinforcing that problems are clustered on specific dates/tables.

Your notes align: ingestion via Export MI, drift due to non-real-time batches, and the need to standardize audit filters/ranges.

What they are trying to fix

Detect and repair missing prior-day inserts reliably (daily audit).

Identify and remediate historical drift (deep dive) on dates where counts diverge.

Handle deletes correctly (they often don’t propagate to the lakehouse/local).

Establish table-appropriate audit/repair patterns (LMTS vs. created-date vs. no-date tables).

Reduce false deltas from timing/filter mismatches between ingestion and validation.

Where they need your help (as Data Engineer)

Reconciliation design: tighten count comparisons so cloud/local/lakehouse use identical filters (company, date window, include/exclude flags).

Automation: parameterize and schedule deep-dive audits (rolling windows) and surgical repair jobs per table/date.

Delete handling: implement/verify the variation-number technique to backfill/reconcile without disruptive deletes; define a safe process for tables lacking LMTS.

Observability: improve logging/alerts (API request, parameters used, counts, retry outcome) and maintain an audit trail of each repair.

Playbooks: document per-table strategy (LMTS vs. created-date vs. Hulk-Smash) and runbooks for weekend catch-up.

Requirements to gather (questions to drive)

Scope & criticality

Which tenants/tables are business-critical (e.g., GL, manufacturing text) and SLOs for drift?

Schema & metadata

For each audited table: presence of LMTS, created date, soft-delete flags, required company filters, multi-company semantics.

Ingestion specifics

Exact Export MI parameters in use (e.g., include archived/corrupt entries, default company), delimiter, paging limits; confirm these match validation queries.

Watermark policy and time window used during loads vs. audits (avoid off-by-hour deltas).

Repair constraints

Approved maintenance windows; rules for using variation number vs. physical delete; max table downtime tolerated.

Environments & ownership

Who owns each dataset and approves reloads; environments covered (PRD only vs. DEV/TRN).

Reporting

Distribution list and cadence for the daily email; where to persist deep-dive results (e.g., TB_Audit, dashboards).

Access you’ll need

Export MI (ION API) credentials and documentation; ability to run SELECT/COUNT with custom filters.

SQL Server (read/write to audit tables like TB_Audit, ability to execute repair scripts).

Lakehouse storage & ETL orchestration access (view pipeline configs, logs, retries).

Job scheduler/ETL tool (to read/modify audit and repair packages).

Monitoring/logs for API calls and ETL (to trace parameterization and failures).

Recommended next steps (pragmatic, low-risk)

Make comparisons apples-to-apples

Standardize a shared parameter object (company, start/end timestamps, include flags) used by both ingestion and audits.

Exclude “today” uniformly and align to [midnight, midnight) windows.

Per-table strategy matrix

Classify all audited tables into LMTS / created-date / no-date; capture the appropriate audit and repair method for each.

Automate deep-dive

Rolling 31-day job per flagged table to locate bad dates; write results to TB_Audit_Detail.

Non-disruptive repairs

Implement the variation-number backfill: tag candidates, reload date window via EVS002, then delete remaining variation=1.

Delete verification

Add a periodic negative-delta check path (separate from the daily email) to quantify suspected missed deletes and queue targeted repairs.

Observability

Log Export MI requests (table, params, timestamp) and store cloud/local counts side-by-side with run IDs; alert only on post-retry non-zero positives.

Dashboard

Simple Power BI/Looker page: today’s flags, 7-day trend, top offending tables, repair backlog & SLA.
