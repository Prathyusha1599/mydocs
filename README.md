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
