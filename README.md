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

import pandas as pd

def get_users_with_managers(graph_token):
    url = "https://graph.microsoft.com/v1.0/users?$select=id,displayName,userPrincipalName,jobTitle,department,manager"
    headers = {"Authorization": f"Bearer {graph_token}"}
    users = []

    next_url = url
    while next_url:
        response = requests.get(next_url, headers=headers)
        response.raise_for_status()
        data = response.json()
        users.extend(data.get("value", []))
        next_url = data.get("@odata.nextLink")

    return pd.DataFrame(users)

users_df = get_users_with_managers(graph_token)
users_df.head()


------------------------------
<inbound>
    <base />

    <!-- Step 1: Extract Bearer token from Authorization header -->
    <set-variable name="access_token" value="@(context.Request.Headers.GetValueOrDefault("Authorization", "").StartsWith("Bearer ") ? context.Request.Headers.GetValueOrDefault("Authorization").Substring(7) : string.Empty)" />

    <!-- Optional: Check if token is missing or empty -->
    <choose>
        <when condition="@(string.IsNullOrEmpty((string)context.Variables["access_token"]))">
            <return-response>
                <set-status code="401" reason="Unauthorized" />
                <set-body>{"error": "Missing or invalid Authorization token."}</set-body>
            </return-response>
        </when>
    </choose>

    <!-- Step 2: Rewrite URI to pass token to backend as query param (if backend still expects it) -->
    <!-- If your backend supports Authorization header, you can skip this -->
    <rewrite-uri template="/data?access_token={access_token}" />

    <!-- Step 3: Optional pagination validation -->
    <validate-parameters>
        <!-- Validate 'page' -->
        <parameter name="page" required="true">
            <validation rule="int" min-value="1" />
        </parameter>
        
        <!-- Validate 'pageSize' -->
        <parameter name="pageSize" required="false">
            <validation rule="int" min-value="1" max-value="100" />
        </parameter>
    </validate-parameters>
</inbound>
------------------------

<inbound>
    <base />

    <!-- Step 1: Extract Bearer token from Authorization header -->
    <set-variable name="access_token" value="@{
        var authHeader = context.Request.Headers.GetValueOrDefault("Authorization", "");
        return authHeader.StartsWith("Bearer ") ? authHeader.Substring(7) : string.Empty;
    }" />

    <!-- Step 2: Check if token is missing or empty -->
    <choose>
        <when condition="@{ return string.IsNullOrEmpty((string)context.Variables["access_token"]); }">
            <return-response>
                <set-status code="401" reason="Unauthorized" />
                <set-body>{"error": "Missing or invalid Authorization token."}</set-body>
            </return-response>
        </when>
    </choose>

    <!-- Step 3: Rewrite URI to include access_token as query param -->
    <rewrite-uri template="/data?access_token={access_token}" />

    <!-- Step 4: Validate pagination parameters -->
    <validate-parameters>
        <!-- Validate 'page' -->
        <parameter name="page" required="true">
            <validation rule="int" min-value="1" />
        </parameter>

        <!-- Validate 'pageSize' -->
        <parameter name="pageSize" required="false">
            <validation rule="int" min-value="1" max-value="100" />
        </parameter>
    </validate-parameters>
</inbound>

-----------------
<inbound>
    <base />

    <!-- Extract Bearer token from Authorization header -->
    <set-variable name="authHeader" value="@(context.Request.Headers.GetValueOrDefault("Authorization", string.Empty))" />
    
    <!-- Extract token by removing 'Bearer ' prefix -->
    <set-variable name="access_token" value="@{
        string header = context.Variables.GetValueOrDefault("authHeader", "") as string;
        return header != null && header.StartsWith("Bearer ") ? header.Substring(7) : string.Empty;
    }" />

    <!-- Reject request if token is missing -->
    <choose>
        <when condition="@((string)context.Variables["access_token"] == string.Empty)">
            <return-response>
                <set-status code="401" reason="Unauthorized" />
                <set-body>{"error": "Missing or invalid Authorization token."}</set-body>
            </return-response>
        </when>
    </choose>

    <!-- Rewrite URI to include token (if backend still expects query param) -->
    <rewrite-uri template="/data?access_token={access_token}" />

    <!-- Validate pagination parameters -->
    <validate-parameters>
        <parameter name="page" required="true">
            <validation rule="int" min-value="1" />
        </parameter>
        <parameter name="pageSize" required="false">
            <validation rule="int" min-value="1" max-value="100" />
        </parameter>
    </validate-parameters>
</inbound>
----------------------------------------

Microsoft Fabric Naming Standards and Guidelines
1. Purpose

This document defines the standard naming conventions for all Fabric artifacts to ensure consistency, clarity, and maintainability across projects.

The goal is to make it easy for any team member—present or future—to quickly identify the source, purpose, and stage of any Fabric object.

2. General Naming Rules
Rule	Description	Example
Use lowercase	Use lowercase letters; avoid camelCase or PascalCase.	✅ pl_m3_stg_ingest_finance
Use underscores	Use _ to separate logical sections.	✅ lh_sales_bronze
Avoid spaces/special characters	No spaces or symbols except _.	❌ Sales-Pipeline ✅ sales_pipeline
Use consistent prefixes	Prefix each artifact by its type.	✅ pl_salesforce_bronze_ingest
Keep names descriptive but concise	Enough info to identify purpose quickly.	✅ ds_sales_gold
Optional versioning	Add _v1, _v2 if multiple versions exist.	✅ pl_anaplan_transform_v2
3. Prefix Standards by Artifact Type
Artifact Type	Prefix	Example	Description
Pipeline	pl_	pl_m3_silver_transform_finance	ETL or orchestration pipeline
Dataflow (Gen2)	df_	df_salesforce_staging	Reusable Power Query transformations
Lakehouse	lh_	lh_hr_bronze	Fabric lakehouse for specific domain
Warehouse	wh_	wh_finance_reporting	Fabric SQL warehouse
Dataset	ds_	ds_marketing_gold	Power BI dataset (semantic model)
Notebook	nb_	nb_data_cleaning_anaplan	Fabric notebook for transformation or ML
Data Pipeline Parameters	prm_	prm_anaplan_monthly_run	Parameter files or datasets used in pipelines
Dataflow Storage Table	tb_	tb_sales_transactions	Physical table in lakehouse or warehouse
Report / Dashboard	rpt_	rpt_finance_summary	Power BI report
Workspace	ws_	ws_enterprise_data_platform	Fabric workspace name
Lakehouse Folder / Zone	stg_, bronze_, silver_, gold_	lh_finance_bronze	Logical data zone naming
4. Naming Convention Pattern

Format:

<Prefix>_<SourceSystem>_<TargetZone>_<Action/Purpose>_<BusinessDomain>


Example:

pl_salesforce_silver_transform_sales

Segment	Meaning
pl	Pipeline artifact
salesforce	Source system
silver	Data zone
transform	Pipeline type or action
sales	Business domain
5. Suggested Naming Examples by Source System
Source System	Example Artifacts
Infor M3	pl_m3_stg_ingest_finance, lh_m3_silver
Anaplan	pl_anaplan_bronze_transform_budget, ds_anaplan_gold
Monday.com	pl_monday_stg_ingest_projects, rpt_monday_project_status
Limble	pl_limble_silver_load_assets, lh_limble_bronze
Fluence	pl_fluence_gold_transform_financials
Salesforce	pl_salesforce_bronze_ingest_crm, ds_salesforce_gold_sales
6. Folder / Zone Structure Standards
Zone	Purpose	Example Folder
staging / raw	Initial ingestion, minimal transformation	/lh_m3_stg/
bronze	Cleaned and standardized data	/lh_anaplan_bronze/
silver	Joined and business-ready tables	/lh_salesforce_silver/
gold	Final analytics layer for reporting	/lh_finance_gold/
7. Example End-to-End Flow
Layer	Artifact	Example Name
Ingestion	Pipeline	pl_m3_stg_ingest_finance
Raw Storage	Lakehouse	lh_m3_stg
Transformation	Notebook	nb_m3_silver_transform_finance
Curated Output	Dataset	ds_finance_gold
Visualization	Report	rpt_finance_summary
8. Future-Proofing Guidelines

Reserve unique codes for new source systems (e.g., sf for Salesforce, an for Anaplan).

Maintain a “Naming Registry” – a shared Excel/SharePoint sheet listing all Fabric artifact names.

Apply automated validation during deployment to enforce naming rules.

Avoid hardcoding names in scripts; use parameters or configuration files.

Keep this document updated as new artifacts or systems are added.

9. Summary of Common Prefixes
Prefix	Artifact Type
pl_	Pipeline
df_	Dataflow
lh_	Lakehouse
wh_	Warehouse
ds_	Dataset
nb_	Notebook
prm_	Parameter
tb_	Table
rpt_	Report
ws_	Workspace
10. Example Naming Across Systems
Source	Artifact	Example Name
Infor M3	Pipeline	pl_m3_bronze_ingest_hr
Anaplan	Notebook	nb_anaplan_silver_transform_budget
Salesforce	Dataset	ds_salesforce_gold_sales
Monday.com	Report	rpt_monday_project_tracker
Limble	Lakehouse	lh_limble_bronze
Fluence	Warehouse	wh_fluence_reporting

-----------------------
10/15

<inbound>
    <base />

    <!-- Token validation -->
    <validate-header name="Authorization" failed-check-httpcode="401" failed-check-error-message="Authorization header is missing or invalid." />
    <choose>
        <when condition="@(context.Request.Headers.GetValueOrDefault("Authorization", "").StartsWith("Bearer "))">
        </when>
        <otherwise>
            <return-response>
                <set-status code="401" reason="Unauthorized" />
                <set-header name="WWW-Authenticate" exists-action="override">
                    <value>Bearer realm="my-api"</value>
                </set-header>
                <set-body>Missing or invalid Bearer token.</set-body>
            </return-response>
        </otherwise>
    </choose>

    <!-- Pagination logic -->
    <set-variable name="page" value="@(context.Request.Url.Query.GetValueOrDefault("page", "1"))" />
    <set-variable name="pageSize" value="@(context.Request.Url.Query.GetValueOrDefault("pageSize", "10"))" />
    <set-variable name="offset" value="@((int.Parse(context.Variables["page"]) - 1) * int.Parse(context.Variables["pageSize"]))" />
    <rewrite-uri template="/your-backend-path?offset={offset}&limit={pageSize}" />
</inbound>

---------
<inbound>
  <base />

  <!-- === Bearer token check (presence + scheme) === -->
  <!-- validate-header ensures header exists and is non-empty -->
  <validate-header name="Authorization"
                   failed-check-httpcode="401"
                   failed-check-error-message="Authorization header is missing."
                   ignore-case="true" />

  <!-- Ensure it is actually a Bearer token -->
  <choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("Authorization", string.Empty).Trim().StartsWith("Bearer ", System.StringComparison.OrdinalIgnoreCase))">
      <!-- If you need real JWT validation, replace this block with <validate-jwt> -->
    </when>
    <otherwise>
      <return-response>
        <set-status code="401" reason="Unauthorized" />
        <set-header name="WWW-Authenticate" exists-action="override">
          <value>Bearer realm="https://abc-fabric.azure-api.net"</value>
        </set-header>
        <set-body>Missing or invalid Bearer token.</set-body>
      </return-response>
    </otherwise>
  </choose>

  <!-- === Pagination (robust parsing + sane bounds) === -->
  <!-- page -->
  <set-variable name="page" value="@{
      var raw = context.Request.Url.Query.GetValueOrDefault("page", "1");
      int p;
      if (!int.TryParse(raw, out p) || p < 1) p = 1;
      return p;
  }" />

  <!-- pageSize with min/max guardrails -->
  <set-variable name="pageSize" value="@{
      var raw = context.Request.Url.Query.GetValueOrDefault("pageSize", "10");
      int s;
      if (!int.TryParse(raw, out s) || s < 1) s = 10;
      // cap to prevent oversized pages (tune as you like)
      if (s > 100) s = 100;
      return s;
  }" />

  <!-- offset = (page-1) * pageSize -->
  <set-variable name="offset" value="@{
      var p = (int)context.Variables["page"];
      var s = (int)context.Variables["pageSize"];
      // checked arithmetic to avoid overflow (paranoid, but safe)
      try { return checked((p - 1) * s); } catch { return 0; }
  }" />

  <!-- === Rewrite to backend with calculated query params === -->
  <!-- Use inline expressions in the template to avoid unresolved {placeholders} -->
  <rewrite-uri template="/workflows/xyz/triggers/zzz/paths/invoke?offset=@(context.Variables["offset"])&limit=@(context.Variables["pageSize"])"
               copy-unmatched-params="true" />
</inbound>

----
<!-- Rewrite + set query params separately to avoid inline @() in template -->
  <rewrite-uri template="/workflows/xyz/triggers/zzz/paths/invoke" copy-unmatched-params="true" />
  <set-query-parameter name="offset" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("offset", 0))</value>
  </set-query-parameter>
  <set-query-parameter name="limit" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("pageSize", 10))</value>
  </set-query-parameter>
</inbound>

----------


<inbound>
  <base />

  <!-- Authorization header present -->
  <validate-header name="Authorization"
                   failed-check-httpcode="401"
                   failed-check-error-message="Authorization header is missing." />

  <!-- Must be a Bearer token -->
  <choose>
    <when condition='@(context.Request.Headers.GetValueOrDefault("Authorization", string.Empty)
                        .Trim()
                        .StartsWith("Bearer ", System.StringComparison.OrdinalIgnoreCase))'>
      <!-- proceed -->
    </when>
    <otherwise>
      <return-response>
        <set-status code="401" reason="Unauthorized" />
        <set-header name="WWW-Authenticate" exists-action="override">
          <value>Bearer realm="https://abc-fabric.azure-api.net"</value>
        </set-header>
        <set-body>Missing or invalid Bearer token.</set-body>
      </return-response>
    </otherwise>
  </choose>

  <!-- Pagination: robust parsing -->
  <set-variable name="page" value='@{
      var raw = context.Request.Url.Query.GetValueOrDefault("page", "1");
      int p;
      if (!int.TryParse(raw, out p) || p < 1) p = 1;
      return p;
  }' />

  <set-variable name="pageSize" value='@{
      var raw = context.Request.Url.Query.GetValueOrDefault("pageSize", "10");
      int s;
      if (!int.TryParse(raw, out s) || s < 1) s = 10;
      if (s > 100) s = 100;  // cap
      return s;
  }' />

  <set-variable name="offset" value='@{
      var p = (int)context.Variables.GetValueOrDefault("page", 1);
      var s = (int)context.Variables.GetValueOrDefault("pageSize", 10);
      try { return checked((p - 1) * s); } catch { return 0; }
  }' />

  <!-- Rewrite + set query params separately to avoid inline @() in template -->
  <rewrite-uri template="/workflows/xyz/triggers/zzz/paths/invoke" copy-unmatched-params="true" />
  <set-query-parameter name="offset" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("offset", 0))</value>
  </set-query-parameter>
  <set-query-parameter name="limit" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("pageSize", 10))</value>
  </set-query-parameter>
</inbound>

----------------------

<inbound>
  <base />

  <!-- Authorization header present -->
  <validate-header name="Authorization"
                   failed-check-httpcode="401"
                   failed-check-error-message="Authorization header is missing." />

  <!-- Must be a Bearer token -->
  <choose>
    <when condition='@(context.Request.Headers.GetValueOrDefault("Authorization", string.Empty)
                        .Trim()
                        .StartsWith("Bearer ", System.StringComparison.OrdinalIgnoreCase))'>
      <!-- proceed -->
    </when>
    <otherwise>
      <return-response>
        <set-status code="401" reason="Unauthorized" />
        <set-header name="WWW-Authenticate" exists-action="override">
          <value>Bearer realm="https://abc-fabric.azure-api.net"</value>
        </set-header>
        <set-body>Missing or invalid Bearer token.</set-body>
      </return-response>
    </otherwise>
  </choose>

  <!-- Pagination: robust parsing -->
  <set-variable name="page" value='@{
      var raw = context.Request.Url.Query.GetValueOrDefault("page", "1");
      int p;
      if (!int.TryParse(raw, out p) || p < 1) p = 1;
      return p;
  }' />

  <set-variable name="pageSize" value='@{
      var raw = context.Request.Url.Query.GetValueOrDefault("pageSize", "10");
      int s;
      if (!int.TryParse(raw, out s) || s < 1) s = 10;
      if (s > 100) s = 100;  // cap
      return s;
  }' />

  <set-variable name="offset" value='@{
      var p = (int)context.Variables.GetValueOrDefault("page", 1);
      var s = (int)context.Variables.GetValueOrDefault("pageSize", 10);
      try { return checked((p - 1) * s); } catch { return 0; }
  }' />

  <!-- Rewrite + set query params separately to avoid inline @() in template -->
  <rewrite-uri template="/workflows/xyz/triggers/zzz/paths/invoke" copy-unmatched-params="true" />
  <set-query-parameter name="offset" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("offset", 0))</value>
  </set-query-parameter>
  <set-query-parameter name="limit" exists-action="override">
    <value>@(context.Variables.GetValueOrDefault("pageSize", 10))</value>
  </set-query-parameter>
</inbound>


