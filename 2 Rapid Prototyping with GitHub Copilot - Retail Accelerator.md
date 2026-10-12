## 2. Rapid Prototyping

Complete the Business and Technical Envisioning whiteboard before starting this exercise. After the whiteboarding session, use the documented business priorities, technical discovery, target solution components, and future-state architecture to build a working prototype of the **Retail Accelerator** with GitHub Copilot.

You will build each block of the **Accelerator Architecture**, in the same order that data flows through it:

| # | Architecture block | What GitHub Copilot builds |
|---|---|---|
| 1 | Source Data | ADLS Gen2 storage account with Customer, Products, Sales, and Finance files |
| 2 | Azure Databricks | Premium workspace, Unity Catalog `retail` catalog, domain schemas and Delta tables |
| 3 | Microsoft Fabric and OneLake | Fabric workspace, `lh_bronze`, `lh_silver`, `lh_gold` lakehouses, OneLake shortcut to source data |
| 4 | Medallion Architecture | Bronze, Silver, and Gold notebooks and the `pl_medallion` pipeline |
| 5 | Mirroring | Mirrored Azure Databricks catalog `mir_retail` and a Gold shortcut |
| 6 | Data Mesh | Fabric domains for Customer, Products, Sales, and Finance |
| 7 | Semantic model, Power BI, and Copilot | `sm_retail` semantic model, `rpt_retail` report, Copilot for Power BI |
| 8 | Fabric Data Agent | `da_retail` agent grounded on the semantic model |
| 9 | Microsoft Purview | Collections per domain, scans of the storage account and Fabric |

![Accelerator Architecture](./Images/Future-State-Architecture.png)

# Prerequisites

> **Important:** Confirm that you have the required software, tools, access, architecture file, and sample data before beginning the exercise.

## Required Software and Tools

- Visual Studio Code
- GitHub
- GitHub Copilot Chat
- Azure CLI
- Bicep CLI
- PowerShell 7 or later
- Python 3.10 or later
- Databricks CLI
- Azure Portal
- Microsoft Fabric
- Microsoft Purview
- Power BI

## Required Access

- An [Azure subscription](https://azure.microsoft.com/) where resources can be deployed.
- **Owner**, or **Contributor** plus **User Access Administrator**, on the subscription or resource group. Copilot needs these roles to create resources and assign data access roles. See [Azure built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles).
- Permission to [register Azure resource providers](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider).
- An existing [Microsoft Fabric capacity](https://learn.microsoft.com/fabric/enterprise/licenses) (F4 or higher), or permission to create one.
- **Fabric administrator** role for the Data Mesh domains and the Purview scan of Fabric.
- **Unity Catalog metastore admin** in Azure Databricks. This role is needed to enable external data access for mirroring.
- **Data Source Administrator** and **Collection Administrator** roles in [Microsoft Purview](https://learn.microsoft.com/purview/purview).
- Ability to sign in to all the services listed above. For sign-in assistance, see [Microsoft account sign-in help](https://support.microsoft.com/en-us/).

## Fabric Tenant Settings (Admin Pre-check)

> **Note:** These settings are usually configured by the lab owner before the workshop. GitHub Copilot cannot change tenant settings. Confirm them in **Fabric → Settings → Admin portal → Tenant settings**.

- Users can create Fabric items
- Copilot and Azure OpenAI Service, including cross-geo processing and storage if your capacity region requires it
- Fabric data agent
- Service principals or users can use Fabric APIs
- Workspace identity creation
- Mirrored Azure Databricks catalog
- Admin API settings for Microsoft Purview (enhanced metadata responses), needed for the Purview scan of Fabric

## Sample Data

Download **Retail_Sample_Data.zip** and extract it into a folder named `data` in an empty working folder. The folder contains:

| File | Rows | Columns |
|---|---|---|
| customers.csv | 200 | customer_id, customer_name, email, city, state, segment, signup_date |
| products.csv | 20 | product_id, product_name, category, unit_price, unit_cost |
| sales.csv | 3,000 | order_id, order_date, customer_id, product_id, quantity, unit_price, discount, net_amount |
| finance.csv | 48 | finance_id, period, category, revenue, cogs, gross_margin |

> **Expected result:** Total sales revenue and total finance revenue both equal **1,587,271.57**. You will use this number to validate every layer.

## Azure Resource Provider Registration

Before deployment, register the resource providers required by the generated templates in PowerShell:

```powershell
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Databricks
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Fabric
az provider register --namespace Microsoft.Purview
```

## Naming Standard

GitHub Copilot uses the following names in every prompt. Do not change them during the exercise, because later prompts reuse earlier resources.

| Item | Name |
|---|---|
| Resource group | `rg-retail-acc` (one region for everything = the Fabric capacity region) |
| Storage account / container | `stretail<unique>` / `retail-source` |
| Databricks workspace / catalog | `dbw-retail` / `retail` |
| Databricks schemas | `customer`, `products`, `sales`, `finance` |
| Fabric workspace | `Retail-Accelerator` |
| Lakehouses | `lh_bronze`, `lh_silver`, `lh_gold` |
| Notebooks / pipeline | `10_bronze_validate`, `20_silver`, `30_gold` / `pl_medallion` |
| Mirrored catalog | `mir_retail` |
| Semantic model / report / agent | `sm_retail` / `rpt_retail` / `da_retail` |

# Rapid prototyping using GitHub Copilot

In this section, you will use GitHub Copilot to turn the future-state architecture from the whiteboarding session into a working prototype. Each component has a GitHub Copilot prompt, steps to monitor the deployment, portal validation steps, and a fix-up prompt for when the validation finds gaps.

### Sign in to GitHub Copilot Chat

Use the following steps to sign in to GitHub Copilot in Visual Studio Code. Complete each authorization prompt in sequence before continuing.

1. Open **Visual Studio Code**, select **File → Open Folder**, and open the working folder that contains the `data` folder.

      ![Visual Studio Code application](./Images/amp14.png)

1. Select **Continue with GitHub** to sign in to GitHub Copilot.

      ![GitHub sign-in page](./Images/amp18.png)

1. On the **Sign in to GitHub** page, enter your **GitHub username** and password, and then select **Sign in**. If prompted, complete the organization single sign-on.

      ![GitHub sign-in form](./Images/amp19.png)

1. Review the authorization request, select **Accept**, and then select **Authorize Visual Studio Code**.

      ![GitHub authorization consent](./Images/amp21.png)

1. When prompted to open Visual Studio Code, select **Open**.

1. Open GitHub Copilot Chat by selecting **Chat** from the **View** menu.

      ![GitHub Copilot Chat panel in Visual Studio Code](./Images/GitHub%20Chat%20Open.png)

1. In GitHub Copilot Chat, select **Models**, and then select **Trust Folder and Continue**.

      ![Trust folder and continue prompt](./Images/amp30.png)

1. Set the mode to **Agent**, and then select the recommended model.

      ![GitHub Copilot model selection](./Images/b7.png)

      > **Note:** If you are unable to select **Models**, wait `2-3 minutes` and check again. If you're still unable to select a model, make sure you are signed in properly.

1. Select **Default permission**, set the permission level to **Allow all**, and then select **Enable**.

      ![GitHub Copilot permission settings](./Images/b8.png)

1. In GitHub Copilot Chat, select the plus sign **+**, and then select **Future-State-Architecture.png** to attach the architecture diagram to the conversation.

      ![Attach future-state architecture to Copilot Chat](./Images/b89.png)

      > **Important:** Keep the same chat session open for all components, so that Copilot reuses the resources it has already created. If you start a new chat, attach the architecture image again.

---

# Environment Setup

Copilot checks your tools, signs you in, and creates the project structure before it deploys anything.

## GitHub Copilot Prompt

```text
Read the attached Accelerator Architecture. You will build it as a working prototype, one component at a time, in later prompts.
Please follow these instructions for environment setup:
1. Check that Azure CLI, Bicep CLI, PowerShell 7, Python 3 and Databricks CLI are installed. Install anything missing.
2. Sign me in with az login, show my subscription, and detect any existing Fabric capacity and Microsoft Purview account.
3. Confirm these resource providers are registered: Microsoft.Storage, Microsoft.Databricks, Microsoft.Compute,
   Microsoft.Network, Microsoft.ManagedIdentity, Microsoft.Fabric, Microsoft.Purview.
4. Check my roles: Azure Owner (or Contributor + User Access Administrator), Fabric administrator, and Databricks metastore admin.
   List clearly what is missing.
5. Create the folders infra, databricks, fabric, purview and docs. Use the CSV files in ./data as the sample data.
6. Use these names for the whole exercise: rg-retail-acc, storage stretail<4 random characters>, container retail-source,
   Databricks dbw-retail, catalog retail, Fabric workspace Retail-Accelerator, lakehouses lh_bronze, lh_silver, lh_gold.
Rules: use Microsoft Learn as the source of truth, never store passwords, keys or tokens in files, and ask me before
creating anything that costs money. If a step can only be done in a portal, give me exact click steps and wait for "done".
Note: Create docs/environment-check.md with the results.
```

1. Paste the prompt into GitHub Copilot Chat, confirm that the architecture image is attached, and select **Send**.

      ![Environment setup prompt](./Images/rp-00-setup-prompt.png)

1. When Copilot runs `az login`, complete the browser sign-in with **<inject key="AzureAdUserEmail">User Email</inject>**, and then select your subscription.

      ![Azure sign-in](./Images/rp-00-az-login.png)

1. Review **docs/environment-check.md**. If a role is missing, ask the lab owner for it before you continue.

      ![Environment check results](./Images/rp-00-env-check.png)

---

# Source Data

The **Source Data** block is an Azure Data Lake Storage Gen2 account. It holds the raw files for the four business domains.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the Source Data block.
Please follow these instructions:
1. Generate parameterized Bicep files for resource group rg-retail-acc and an ADLS Gen2 storage account
   stretail<unique> (hierarchical namespace enabled, TLS 1.2, no public blob access, shared key access disabled).
   Use the same region as my Fabric capacity.
2. Run az bicep build with zero warnings, show me the what-if result, and ask before deploying.
3. Create container retail-source with folders customers, products, sales and finance, and upload the matching CSV
   from ./data into each folder using --auth-mode login.
4. Assign me Storage Blob Data Contributor on the storage account.
5. List the uploaded files as proof.
Note: Add the deployment steps and validation results to docs/deployment-guide.md.
```

1. Paste the prompt and select **Send**.

1. When Copilot asks for confirmation to create the resource group and storage account, review the what-if output and select **Yes**.

      ![Storage deployment confirmation](./Images/rp-01-storage-confirm.png)

1. Wait for the deployment to complete (approximately `3–5` minutes), and then select **Keep** to retain the generated files.

### Validate in the Azure portal

1. In the [Azure portal](https://portal.azure.com), open **Resource groups → rg-retail-acc**, and confirm that the storage account is present.

      ![Resource group with storage account](./Images/rp-01-rg.png)

1. Open the storage account and select **Settings → Configuration**. Confirm that **Hierarchical namespace** is **Enabled**.

1. Select **Containers → retail-source**. Confirm that the four folders each contain their CSV file.

      ![Source data folders](./Images/rp-01-container.png)

---

# Azure Databricks

**Azure Databricks** is the upstream data-processing component. It stores curated domain data in Unity Catalog, and Fabric mirrors that data into OneLake.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the Azure Databricks component.
Please follow these instructions:
1. Generate a Bicep module for Azure Databricks workspace dbw-retail (Premium tier) in rg-retail-acc, in the same region.
   Ask before deploying.
2. Using the Databricks CLI and the SQL Statement Execution API on a serverless SQL warehouse:
   - create catalog retail (default storage) and schemas customer, products, sales, finance
   - create a Unity Catalog volume retail.sales.landing and upload the four CSV files from ./data
   - create Delta tables retail.customer.customers, retail.products.products, retail.sales.sales_curated and
     retail.finance.finance using read_files with header and inferSchema enabled
3. Validate row counts: customers 200, products 20, sales_curated 3000, finance 48.
4. Prepare Unity Catalog for Fabric mirroring: enable "External data access" on the metastore and run
   GRANT EXTERNAL USE SCHEMA ON CATALOG retail TO my user. If I am not a metastore admin, stop and give me the steps.
5. Explain in simple words how data moves from the files into Databricks.
Note: Update docs/deployment-guide.md with the generated files, commands, validation results and any manual steps.
```

1. Paste the prompt and select **Send**.

1. When Copilot asks for confirmation to deploy the Premium workspace, select **Yes**. The deployment takes approximately `5–8` minutes.

      ![Databricks deployment confirmation](./Images/rp-02-dbw-confirm.png)

1. If the Databricks CLI opens a browser window for authentication, sign in with your lab account.

1. Wait for Copilot to report the row counts, and then select **Keep**.

### Validate in Azure Databricks

1. In the Azure portal, open **dbw-retail**, and then select **Launch Workspace**.

1. Select **Catalog**, expand **retail**, and confirm that the four schemas and tables are present.

      ![Unity Catalog tables](./Images/rp-02-catalog.png)

1. Open **SQL Editor**, run the following query, and confirm that it returns 200, 20, 3000 and 48.

      ```sql
      SELECT 'customers', COUNT(*) FROM retail.customer.customers UNION ALL
      SELECT 'products', COUNT(*) FROM retail.products.products UNION ALL
      SELECT 'sales', COUNT(*) FROM retail.sales.sales_curated UNION ALL
      SELECT 'finance', COUNT(*) FROM retail.finance.finance;
      ```

      ![Row count validation](./Images/rp-02-counts.png)

1. Select **Catalog → ⚙ → Metastore**, and confirm that **External data access** is enabled.

      ![External data access enabled](./Images/rp-02-external-access.png)

      > **Note:** If you are not a metastore admin, ask the lab owner to enable this setting and run the grant. Mirroring will fail without it.

---

# Microsoft Fabric and OneLake

**OneLake** is the unified data foundation of Microsoft Fabric. It is part of Fabric, not a separate storage account. In this step, Copilot creates the Fabric workspace and the three medallion lakehouses, and it connects Bronze to the source data with a zero-copy OneLake shortcut.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the Microsoft Fabric and OneLake foundation using the Fabric REST APIs.
Please follow these instructions:
1. Use my existing Fabric capacity. If none exists, generate Bicep for a Fabric capacity (F4) in the same region and ask
   before deploying, because billing starts immediately.
2. Create workspace Retail-Accelerator on the capacity and assign me as Admin.
3. Create lakehouses lh_bronze, lh_silver and lh_gold, with lakehouse schemas disabled.
4. Provision the workspace identity, assign it Storage Blob Data Reader on the storage account, and create a cloud
   connection to ADLS Gen2 that uses workspace identity authentication.
5. Create a OneLake shortcut named retail-source under lh_bronze Files that points to the retail-source container.
6. Handle long-running operations (HTTP 202) by polling the Location header. Make every script safe to re-run.
7. List the workspace items and the shortcut contents as proof.
Note: Explain in docs/deployment-guide.md that OneLake is the storage layer of Fabric and the shortcut avoids copying data.
```

1. Paste the prompt and select **Send**.

1. If Copilot asks you to confirm the Fabric capacity deployment, select **Yes**. If a capacity already exists, Copilot reuses it.

      ![Fabric capacity confirmation](./Images/rp-03-capacity.png)

1. Wait for Copilot to finish, and then select **Keep**.

### Validate in Microsoft Fabric

1. Open [Microsoft Fabric](https://app.fabric.microsoft.com), select **Workspaces**, and open **Retail-Accelerator**.

1. Confirm that **lh_bronze**, **lh_silver** and **lh_gold** are present.

      ![Fabric workspace with lakehouses](./Images/rp-03-workspace.png)

1. Open **lh_bronze**, expand **Files**, and confirm that the **retail-source** shortcut shows the four folders.

      ![OneLake shortcut to source data](./Images/rp-03-shortcut.png)

      > **Note:** If the shortcut is missing or shows an access error, send the following prompt:
      >
      > ```text
      > The lh_bronze shortcut retail-source is missing or shows an access error. Check the workspace identity role
      > assignment on the storage account, wait for the role to propagate, recreate the connection and shortcut, and
      > show me the shortcut contents.
      > ```

---

# Medallion Architecture

The **Medallion Architecture** organizes data into three layers:

- **Bronze:** raw data, exactly as received.
- **Silver:** cleaned, deduplicated, and typed data.
- **Gold:** business-ready dimension and fact tables.

Notebooks transform the data, and a pipeline runs the notebooks in order.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the Medallion Architecture (Bronze, Silver, Gold) in the Retail-Accelerator
workspace using the Fabric REST APIs.
Please follow these instructions:
1. Create notebook 10_bronze_validate with lh_bronze as the default lakehouse. It reads
   Files/retail-source/<table>/<table>.csv for customers, products, sales and finance, saves each as a Delta table
   in lh_bronze with the same name, and prints row counts and schemas.
2. Create notebook 20_silver with lh_silver as the default lakehouse. It reads the Bronze tables, removes duplicates on
   the business keys, converts dates to date type, money columns to decimal(14,2), discount to decimal(4,2) and quantity
   to int, keeps only rows with quantity > 0, and writes silver_customers, silver_products, silver_sales and
   silver_finance (overwrite with overwriteSchema).
3. Create notebook 30_gold with lh_gold as the default lakehouse. It creates dim_customer, dim_product, fact_finance and
   fact_sales. fact_sales = silver_sales joined to products for unit_cost, plus cost_amount = quantity * unit_cost
   (decimal 14,2) and order_month = yyyy-MM. Print orphan customers, orphan products, total sales revenue and total
   finance revenue.
4. Create data pipeline pl_medallion with three Notebook activities, 10_bronze_validate -> 20_silver -> 30_gold,
   connected on success.
5. Run the pipeline, poll until it finishes, and report the status of each activity.
6. Validate: Bronze counts 200/20/3000/48, orphan keys 0 and 0, and sales revenue = finance revenue = 1587271.57.
Note: Save the notebook source files in the fabric folder and add the validation results to docs/post-deployment-validation.md.
```

1. Paste the prompt and select **Send**.

1. Monitor the chat while Copilot creates the notebooks and the pipeline, and then runs the pipeline. The pipeline run takes approximately `5–10` minutes.

      > **Note:** If Copilot asks you to **Continue to iterate**, select **Continue** so that it can finish the run and resolve any errors.

1. When Copilot shows the validation summary, review the results and select **Keep**.

      ![Medallion deployment summary](./Images/rp-04-summary.png)

### Validate in Microsoft Fabric

1. In the **Retail-Accelerator** workspace, confirm that the three notebooks and **pl_medallion** are present.

1. Open **pl_medallion** and confirm that the three activities are chained in order.

      ![Medallion pipeline canvas](./Images/rp-04-pipeline.png)

1. Select **Run history** (or the **Output** tab), and confirm that every activity shows **Succeeded**.

      ![Successful pipeline run](./Images/rp-04-run.png)

1. Open **lh_silver** and **lh_gold**, and confirm the tables:

   - **lh_silver:** `silver_customers`, `silver_products`, `silver_sales`, `silver_finance`
   - **lh_gold:** `dim_customer`, `dim_product`, `fact_sales`, `fact_finance`

      ![Gold tables](./Images/rp-04-gold.png)

1. Open **30_gold** and review the output of the last cell. You should see **0 / 0 / 1587271.57 / 1587271.57**.

      ![Gold validation output](./Images/rp-04-validation.png)

      > **Note:** If any table is missing or the revenue numbers don't match, send the following prompt:
      >
      > ```text
      > The medallion validation failed: <describe what is missing or the numbers you see>. Open the failed pipeline
      > activity, read the notebook error, fix the notebook, re-run pl_medallion, and show me the validation output again.
      > ```

---

# Mirroring

**Mirroring** is the dotted path in the architecture from Azure Databricks to Gold. Fabric reads the Unity Catalog tables in place, with no copy and no ETL, and makes them available in OneLake.

## GitHub Copilot Prompt

```text
Read the attached architecture and implement the Mirroring path from Azure Databricks into Microsoft Fabric and Gold.
Please follow these instructions:
1. Check on Microsoft Learn that a Mirrored Azure Databricks catalog is supported for my region, and confirm that
   External data access and EXTERNAL USE SCHEMA are in place on catalog retail.
2. The Azure Databricks connection uses Organizational account (OAuth) sign-in, which the Fabric REST API cannot create.
   Give me exact portal steps to create a connection named conn-dbw-retail, then wait for me to type "done".
3. Read the connection ID with the Fabric REST API, and create MirroredAzureDatabricksCatalog mir_retail on catalog
   retail with all schemas and auto sync enabled.
4. Wait for the sync, query the SQL analytics endpoint and confirm that sales.sales_curated has 3000 rows and total
   net_amount 1587271.57, matching Databricks.
5. Create a OneLake shortcut named sales_curated in lh_gold Tables that points to the mirrored sales table.
6. If mirroring is blocked, explain the exact reason. Do not replace it with another method and call it mirroring.
Note: Document the source, target, connection, validation results and manual steps in docs/deployment-guide.md.
```

1. Paste the prompt and select **Send**.

1. When Copilot stops for the connection, follow its steps. They match these:

   1. In Fabric, select **⚙ Settings → Manage connections and gateways → + New**.
   1. Select **Cloud**, enter the connection name **conn-dbw-retail**, and then select **Azure Databricks workspace** as the connection type.
   1. Enter the workspace URL (`https://adb-xxxx.azuredatabricks.net`), select **OAuth 2.0 / Organizational account**, and then select **Edit credentials** and sign in.
   1. Select **Create**.

      ![Databricks connection](./Images/rp-05-connection.png)

1. Type **done** in GitHub Copilot Chat. Copilot creates **mir_retail** and validates it. When it finishes, select **Keep**.

### Validate in Microsoft Fabric

1. In **Retail-Accelerator**, open **mir_retail** and confirm that the schemas show as **Running** or synchronized.

      ![Mirrored catalog status](./Images/rp-05-mirror.png)

1. Open the **SQL analytics endpoint** of **mir_retail**, run the following query, and confirm the result **3000 / 1587271.57**.

      ```sql
      SELECT COUNT(*) AS orders, SUM(net_amount) AS revenue FROM sales.sales_curated;
      ```

      ![Mirrored data query](./Images/rp-05-query.png)

1. Open **lh_gold → Tables**, and confirm that the **sales_curated** shortcut is present.

      ![Gold shortcut to mirrored table](./Images/rp-05-gold-shortcut.png)

---

# Data Mesh for Multi Domains

The **Data Mesh** groups data by business domain, so that each domain owns its data products. Fabric domains represent Customer, Products, Sales, and Finance.

## GitHub Copilot Prompt

```text
Read the attached architecture and implement the Data Mesh for Customer, Products, Sales and Finance.
Please follow these instructions:
1. Check whether I am a Fabric administrator.
2. If I am, use the Fabric Admin Domains API to create domains Customer, Products, Sales and Finance with short
   descriptions, and assign workspace Retail-Accelerator to the Sales domain.
3. If I am not, stop and give me the exact Admin portal steps, and record this as a manual step.
4. Map each Gold table to its domain (dim_customer -> Customer, dim_product -> Products, fact_sales -> Sales,
   fact_finance -> Finance) in docs/data-mesh.md.
```

1. Paste the prompt and select **Send**. Review the result and select **Keep**.

### Validate in Microsoft Fabric

1. Select **⚙ Settings → Admin portal → Domains**, and confirm that the four domains are present.

      ![Fabric domains](./Images/rp-06-domains.png)

1. Open the **Sales** domain and confirm that **Retail-Accelerator** is assigned.

      ![Workspace assigned to domain](./Images/rp-06-assign.png)

      > **Note:** Creating domains requires the Fabric administrator role. If you don't see the **Domains** menu, ask the lab owner to complete this step.

---

# Semantic Model, Power BI, and Copilot for Power BI

The **semantic model** turns the Gold tables into business measures. **Power BI** reports visualize the measures, and **Copilot for Power BI** answers questions about them in natural language.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the semantic model and Power BI layer using the Fabric REST APIs.
Please follow these instructions:
1. Create semantic model sm_retail in Direct Lake mode on lh_gold using a TMDL definition, with tables dim_customer,
   dim_product, fact_sales and fact_finance.
2. Add relationships (many-to-one, single direction): fact_sales[customer_id] -> dim_customer[customer_id] and
   fact_sales[product_id] -> dim_product[product_id].
3. Add measures on fact_sales:
   Net Revenue = SUM(fact_sales[net_amount])
   Total Orders = DISTINCTCOUNT(fact_sales[order_id])
   Active Customers = DISTINCTCOUNT(fact_sales[customer_id])
   Gross Margin % = DIVIDE([Net Revenue] - SUM(fact_sales[cost_amount]), [Net Revenue])
4. Refresh the model and validate it with a DAX query through the executeQueries API. Net Revenue must equal 1587271.57.
5. Create report rpt_retail bound to sm_retail using a PBIR definition, with four visuals: line chart (order_month,
   Net Revenue), clustered bar chart (dim_product[category], Net Revenue), card (Gross Margin %) and table
   (customer_name, Net Revenue, Top 10).
6. If report creation fails after 3 attempts, create a report bound to sm_retail and give me click steps for the visuals.
7. Give me three questions to ask Copilot for Power BI, with the expected answers calculated with DAX.
Note: Add the model design, measures and validation results to docs/post-deployment-validation.md.
```

1. Paste the prompt and select **Send**. Review the result and select **Keep**.

      ![Semantic model summary](./Images/rp-07-summary.png)

### Validate in Power BI

1. In **Retail-Accelerator**, open **sm_retail**, and then select **Open data model**. Confirm the two relationships and four measures.

      ![Semantic model diagram](./Images/rp-07-model.png)

1. Open **rpt_retail** and confirm that the four visuals display data.

      ![Retail report](./Images/rp-07-report.png)

1. Select **Edit**, open **Copilot**, and ask the questions that Copilot gave you, for example *"Summarize revenue trends by category"*. Compare the answers with the expected values.

      ![Copilot for Power BI](./Images/rp-07-copilot.png)

      > **Note:** Copilot for Power BI can only be tested in the portal. If the **Copilot** button is unavailable, confirm the Copilot tenant setting and that the workspace is on an F-SKU capacity.

---

# Fabric Data Agent

The **Fabric Data Agent** lets business users ask questions about retail data in natural language. It is grounded on the `sm_retail` semantic model.

## GitHub Copilot Prompt

```text
Read the attached architecture and build the Fabric Data Agent using the Fabric REST APIs.
Please follow these instructions:
1. Create data agent da_retail in Retail-Accelerator with semantic model sm_retail as the data source.
2. Add these agent instructions: "Answer retail business questions using sm_retail. Revenue means the Net Revenue
   measure. Months are in order_month (yyyy-MM). Categories are Electronics, Apparel, Home and Grocery.
   Round numbers to 2 decimals."
3. Add example questions: total net revenue; category with the highest revenue; top 5 customers by revenue;
   active customers in 2026-03.
4. Publish the agent.
5. Calculate the expected answer for each example question with DAX, and save the answers to
   docs/evidence/agent-expected.md so I can compare them with the agent in the portal.
```

1. Paste the prompt and select **Send**. Review the result and select **Keep**.

### Validate in Microsoft Fabric

1. In **Retail-Accelerator**, open **da_retail** and confirm that **sm_retail** is listed as the data source.

      ![Data agent configuration](./Images/rp-08-agent.png)

1. Ask each example question and compare the answer with **docs/evidence/agent-expected.md**.

      ![Data agent answer](./Images/rp-08-answer.png)

      > **Note:** If an answer is wrong, send the following prompt:
      >
      > ```text
      > The data agent answered "<question>" with "<answer>", but the expected answer is "<expected>". Improve the agent
      > instructions and example questions so it uses the correct measure and filter, publish it again and tell me what changed.
      > ```

1. Select **Share** and give a test user access to the agent.

---

# Microsoft Purview

**Microsoft Purview** provides the unified catalog, data discovery, and data governance across the solution.

## GitHub Copilot Prompt

```text
Read the attached architecture and configure Microsoft Purview governance.
Please follow these instructions:
1. Reuse the existing Microsoft Purview account. If none exists, generate Bicep for one in rg-retail-acc and ask before deploying.
2. Assign the Purview managed identity Storage Blob Data Reader on the storage account.
3. Using the Purview Data Map REST APIs, create collections Customer, Products, Sales and Finance.
4. Register the ADLS Gen2 storage account as a data source and run a scan using the managed identity.
5. Register Microsoft Fabric (same tenant) as a data source and run a scan, only if the Fabric admin API settings for
   Purview are enabled. Otherwise, stop and give me the manual steps.
6. Wait for the scans to complete, and confirm that the customers asset and its schema can be found in the catalog.
Note: Document the configuration, validation results and manual steps in docs/purview.md.
```

1. Paste the prompt and select **Send**. If Copilot asks to deploy a Purview account, select **Yes**.

1. Wait for the scans to complete (approximately `5–15` minutes), and then select **Keep**.

### Validate in Microsoft Purview

1. Open the [Microsoft Purview portal](https://purview.microsoft.com), and then select **Data Map → Domains / Collections**. Confirm the four collections.

      ![Purview collections](./Images/rp-09-collections.png)

1. Select **Data sources** and confirm that the storage account and Fabric are registered, and that the last scan shows **Completed**.

      ![Purview scans](./Images/rp-09-scans.png)

1. Search the catalog for **customers**, open the asset, and review its schema.

      ![Purview asset schema](./Images/rp-09-asset.png)

1. In Fabric, select **⚙ Settings → Microsoft Purview hub** to review the governance overview.

---

# Final Validation and Documentation

## GitHub Copilot Prompt

```text
Read the attached architecture and confirm that every block is built and working.
Please follow these instructions:
1. For each block (Source Data, Azure Databricks, Mirroring, OneLake, Bronze, Silver, Gold, Data Mesh, semantic model,
   Power BI, Copilot for Power BI, Fabric Data Agent, Microsoft Purview), show the deployed item, the evidence and the
   status: Completed, Manual step done by user, or Not completed (with the reason).
2. Finish docs/post-deployment-validation.md and docs/manual-steps.md.
3. Create docs/README-non-technical.md that explains the solution in plain language:
   Source -> Databricks -> Mirroring -> OneLake Bronze/Silver/Gold -> semantic model -> Power BI, Copilot and Data Agent,
   governed by Purview.
4. Tell me how to pause the Fabric capacity and delete rg-retail-acc, but do not run these commands.
```

1. Paste the prompt and select **Send**. Review the status table.

      ![Final validation summary](./Images/rp-10-final.png)

1. In **Retail-Accelerator**, confirm that the workspace shows every item: three lakehouses, three notebooks, `pl_medallion`, `mir_retail`, `sm_retail`, `rpt_retail` and `da_retail`.

      ![Final workspace](./Images/rp-10-workspace.png)

## Steps that require manual action

| Step | Why it is manual |
|---|---|
| GitHub Copilot sign-in, trust folder, model, **Allow all**, attach image | Interactive Visual Studio Code setup |
| `az login`, Databricks CLI sign-in, MFA prompts | Interactive sign-in |
| Approving billable deployments | Safety confirmation |
| Fabric tenant settings | Requires Fabric administrator; usually pre-configured by the lab owner |
| Unity Catalog external data access and grant | Requires metastore admin |
| Azure Databricks connection for mirroring | OAuth connections can be created only in the portal |
| Fabric domains | Requires Fabric administrator |
| Purview scan of Fabric prerequisites | Requires Fabric and Purview admin settings |
| Testing Copilot for Power BI and the Data Agent, sharing the agent | Portal-only interaction |
| Report layout polish | Visual design choices |

## Clean up

> **Important:** The Fabric capacity bills while it is running. Clean up when you finish the exercise, unless the lab environment does it for you.

1. In the Azure portal, open the Fabric capacity and select **Pause**.
1. Open **Resource groups → rg-retail-acc** and select **Delete resource group**.

### Congratulations!

You have used GitHub Copilot to turn the Accelerator Architecture into a working Retail prototype. It covers source data, Azure Databricks, Mirroring, a Fabric medallion lakehouse on OneLake, Data Mesh domains, a semantic model with Power BI and Copilot, a Fabric Data Agent, and Microsoft Purview governance.
