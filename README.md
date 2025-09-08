
Documentation to apply the CLS in fabric Lakehouse tables.
https://learn.microsoft.com/en-us/fabric/onelake/security/get-started-data-access-roles
Could you go through it and other similar sources and provide me a detailed guide with steps , pre-requisites, limitations for the above requirement
To apply column-level security (CLS) in Microsoft Fabric Lakehouse tables, the recommended approach is to use the SQL analytics endpoint or Warehouse capabilities, as direct CLS within OneLake table files is not natively supported. The steps, prerequisites, and limitations for enabling this scenario are detailed below, leveraging both Microsoft's documentation and reliable technical sources.
Key Prerequisites
•	Fabric workspace with active capacity or trial. 
•	At least Admin, Member, or Contributor rights on the target workspace or SQL analytics endpoint. 
•	Data must reside in a Fabric Lakehouse and be accessible via a SQL analytics endpoint or Fabric Warehouse (CLS applies at this layer, not on raw OneLake files). 
Step-by-Step Guide
1. Set Up Workspace and Access
•	Log into Microsoft Fabric with an account having elevated access (Admin/Member/Contributor). 
•	Navigate to the workspace, then choose the Warehouse or configure a SQL analytics endpoint on your Lakehouse.
2. Define Data Access Roles (for OneLake Folder-Level—Optional)
•	You may use OneLake data access roles for folder-level security if required; this is managed via “Manage OneLake data access” in the Lakehouse menu.
•	Note: This folder-level access does not by itself provide column-level security. 
3. Assign Column-Level SELECT Permissions
•	Within the Warehouse or SQL analytics endpoint, identify:
o	The table to secure
o	Columns to restrict
o	User/group/role to assign permissions to
•	Use the following T-SQL command:
GRANT SELECT ON YourSchema.YourTable (Column1, Column2, Column3) TO [UserOrRole];

•	Replace:
o	YourSchema with the schema name (often dbo)
o	YourTable with your target table
o	The column list with the columns the role/user should access
o	[UserOrRole] with the user, group, or role’s name (preferably security groups for manageability). 
•	To deny access to certain columns (if needed for granular security):
DENY SELECT ON YourSchema.YourTable (SensitiveColumn) TO [UserOrRole];

•	Repeat for each table and user/group combination as required.
4. Test and Validate
•	Have target users connect via supported applications (e.g., Power BI, SQL tools). Users without explicit column permissions will receive errors when querying restricted columns. 
•	CLS applies at the database tier—all tools and API consumers (including Direct Query mode in Power BI) respect restrictions. 
Limitations and Considerations
•	Direct CLS is not supported for files/folders in OneLake accessed outside the SQL endpoint (e.g., via Spark, file APIs). 
•	Only Microsoft Entra authenticated users are supported for granular column permissions.
•	Out-of-the-box UI for CLS is evolving; most configuration requires T-SQL.
•	Admins, Members, Contributors with full workspace rights can override data access roles for folder-level controls but will be restricted by CLS at the SQL layer. 
•	OneLake folder-level security (OneLake Data Access Roles) is designed for folder/table visibility and read permissions, not column masking. 
•	Power BI in Direct Lake mode falls back to Direct Query when CLS is present, to ensure restrictions are enforced. 
Summary Table: Capabilities
Capability	Folder/File (OneLake)	SQL Analytics Endpoint/Warehouse
Object/Table Security	Yes[1]
Yes[3]

Column-Level Security	No[6]
Yes[2][3][5]

UI Security Management	Yes (folders)	Partial (mostly T-SQL)
Entra ID Group Assignment	Yes	Yes
Affects Direct Lake Mode	No	Yes (enforces via Direct Query)


fabric data --------------------------------------------------------------


Query the Fabric Lakehouse Data Outside the Fabric

To access Microsoft Fabric Lakehouse data in an external application using the SQL analytics endpoint, you need to follow specific implementation steps, fulfill prerequisites, and be prepared for several common blockers. Below is a comprehensive guide based on authoritative sources and direct usage examples[1][2][3].
Direct Answer
You can connect external applications to Fabric Lakehouse data using the SQL analytics endpoint via ODBC/JDBC drivers (e.g., Python's pyodbc), provided you have the endpoint’s connection string and the right authentication (Service Principal or user credentials)[1][2]. Blocking issues often include workspace permissions, endpoint provisioning delays, and authentication configuration[4][5].
 
Implementation Guide
Prerequisites
•	Fabric Premium Workspace access: You must have a Fabric workspace with Premium or higher capacity[3][6].
•	Lakehouse created: Your data must be stored in a Fabric Lakehouse, which automatically generates a SQL analytics endpoint[2][3].
•	SQL analytics endpoint ready: Database tables (typically Delta tables) should exist, and SQL endpoint provisioning should be successful.
•	Python and required libraries: Latest Python, Microsoft ODBC Driver 18 or 17 for SQL Server, and pyodbc library[1].
•	Workspace permissions: Contributor or higher access, Service Principal membership for automated access[2][4].
•	Authentication: Either an Azure Active Directory Service Principal or user credentials with proper privileges.
 
Step-by-Step Implementation
1.	Get SQL Connection String
o	Locate the endpoint URL in Lakehouse Details/Settings (looks like xxxx.datawarehouse.fabric.microsoft.com)[2][7].
2.	Install Required Python Components
o	Install/upgrade Python and pip.
o	pip install pyodbc
o	Download Microsoft ODBC Driver for SQL Server (Windows, Mac, Linux supported)[1].
3.	Prepare Authentication
o	For programmatic access, use Active Directory authentication with Service Principal. Ensure the principal is given access (Member + Build & Read permissions on workspace and dataset)[4].
o	Enable “Service principals can call Fabric APIs” in tenant settings; enable XMLA Read/Write if using semantic models[4].
4.	Code Example (Python)
import pyodbc

connection_string = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER={your_sql_analytics_endpoint};"
    "DATABASE={lakehouse_name};"
    "UID={service_principal_client_id};"
    "PWD={service_principal_secret};"
    "Authentication=ActiveDirectoryServicePrincipal"
)

cnxn = pyodbc.connect(connection_string)
cursor = cnxn.cursor()
cursor.execute("SELECT TOP 100 * FROM [your_table]")
for row in cursor.fetchall():
    print(row)

o	JDBC connection also supported for other environments[2].
 

 
1.	https://www.linkedin.com/pulse/programmatically-query-microsoft-fabric-lakehouse-brice-baro     
2.	https://docs.ataccama.com/one/latest/sources/lakehouse-sql-analytics-endpoint.html        
3.	https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint     
4.	https://www.reddit.com/r/MicrosoftFabric/comments/1m0ume7/how_get_data_from_a_fabric_lakehouse_using/       
5.	https://www.reddit.com/r/MicrosoftFabric/comments/1l4sipp/updated_delays_in_synchronising_the_lakehouse/   
6.	https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse   
7.	https://radacad.com/microsoft-fabric-lakehouse/ 
8.	https://learn.microsoft.com/en-us/fabric/database/sql/query-sql-analytics-endpoint   
9.	https://learn.microsoft.com/en-us/fabric/governance/external-data-sharing-overview    
10.	https://learn.microsoft.com/en-us/fabric/data-warehouse/get-started-lakehouse-sql-analytics-endpoint 
11.	https://learn.microsoft.com/en-us/fabric/data-warehouse/sql-analytics-endpoint-performance 
12.	https://www.reddit.com/r/MicrosoftFabric/comments/1bs7886/programmatically_query_sql_analytics_endpoint/ 
13.	https://www.youtube.com/watch?v=B2x6ggpKFqU 
14.	https://datacrafters.io/secure-external-data-sharing-microsoft-fabric/ 
15.	https://robkerr.ai/query-fabric-table-csharp/ 
16.	https://intellifysolutions.com/blog/microsoft-fabric-lakehouse-vs-data-warehouse/ 
17.	https://www.youtube.com/watch?v=toTKGYwr278 
18.	https://www.youtube.com/watch?v=TeutxliZVRU 
19.	https://community.qlik.com/t5/Connectivity-Data-Prep/Microsoft-Fabric-Lakehouse-amp-Qlik-Cloud-Analytics/td-p/2442001 
20.	https://campus.datacamp.com/courses/introduction-to-microsoft-fabric/working-with-lakehouses?ex=5 
21.	https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse      
22.	https://community.fabric.microsoft.com/t5/Data-Warehouse/Lakehouse-under-PPu/m-p/4274830        
23.	https://www.red-gate.com/simple-talk/databases/sql-server/bi-sql-server/choosing-between-the-lakehouse-and-warehouse-in-microsoft-fabric/       
24.	https://www.serverlesssql.com/enabling-microsoft-fabric-60-day-trial/   
25.	https://www.youtube.com/watch?v=TeutxliZVRU 
26.	https://www.azureguru.net/demystifying-roles-access-in-microsoft-fabric-lake-house 
27.	https://learn.microsoft.com/en-us/fabric/data-warehouse/get-started-lakehouse-sql-analytics-endpoint 
28.	https://learn.microsoft.com/en-us/fabric/security/security-managed-private-endpoints-overview 
29.	https://www.reddit.com/r/MicrosoftFabric/comments/1iofr2n/clarification_on_direct_lake_mode_and_sql/ 
30.	https://radacad.com/microsoft-fabric-lakehouse/ 
31.	https://www.reddit.com/r/MicrosoftFabric/comments/1ecuszy/fabric_lakehouse_tables_not_showing_in_sql/ 
32.	https://intellifysolutions.com/blog/microsoft-fabric-lakehouse-vs-data-warehouse/ 
33.	https://data-marc.com/2023/05/24/overcoming-the-fear-exploring-microsoft-fabric-from-a-power-bi-angle/ 
34.	https://radacad.com/what-is-the-xmla-endpoint-for-power-bi-and-why-should-i-care/ 
35.	https://community.powerbi.com/t5/Power-Query/Connect-to-Datasets-in-PowerBI-via-SSMS/td-p/603247 
36.	https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/ 
37.	https://microsoftlearning.github.io/DP-900T00A-Azure-Data-Fundamentals/Instructions/Labs/dp900-04b-fabric-lake-lab.html 
38.	https://docs.ataccama.com/one/latest/sources/lakehouse-sql-analytics-endpoint.html 
39.	https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint 
40.	https://www.c-sharpcorner.com/article/managed-private-endpoints-in-microsoft-fabric2/ 


 



