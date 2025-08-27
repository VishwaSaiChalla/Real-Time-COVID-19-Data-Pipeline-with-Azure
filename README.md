# COVID-19 Data Reporting Pipeline - Azure Data Factory

A comprehensive Azure Data Factory (ADF) solution for ingesting, processing, and analyzing COVID-19 data from European Centre for Disease Prevention and Control (ECDC) and other European sources.

## 🏗️ Architecture Overview

This project implements a complete ETL (Extract, Transform, Load) pipeline that:
- **Ingests** COVID-19 data from multiple European sources
- **Transforms** raw data using Azure Data Factory dataflows
- **Loads** processed data into Azure Data Lake and SQL Database
- **Automates** execution using triggers and dependencies

## 📊 Data Sources

### Primary Data Sources:
1. **ECDC (European Centre for Disease Prevention and Control)**
   - Cases and deaths data
   - Hospital admissions data
   - Testing data

2. **European Statistical Data**
   - Population demographics by age group
   - Country lookup data
   - Date dimension data

## 🔧 Infrastructure Components

### Azure Services Used:
- **Azure Data Factory**: Orchestration and data movement
- **Azure Data Lake Storage Gen2**: Raw and processed data storage
- **Azure Blob Storage**: Configuration and lookup data
- **Azure SQL Database**: Final data warehouse
- **HDInsight**: On-demand processing cluster

### Factory Configuration:
- **Name**: `covidreportingadfchalla`
- **Location**: East US
- **Identity**: System Assigned Managed Identity

## 📁 Project Structure

```
📦 Azure_Data_Factory_Covid19
├── 📂 dataflow/              # Data transformation logic
│   ├── df_transform_cases_deaths.json
│   ├── df_transform_hospital_admissions.json
│   └── df_transform_population_data.json
├── 📂 dataset/               # Data source and sink definitions
│   ├── 📁 lookup/           # Reference data
│   ├── 📁 raw/              # Source data definitions
│   ├── 📁 processed/        # Transformed data definitions
│   └── 📁 sqlize/           # SQL database definitions
├── 📂 linkedService/         # Connection configurations
├── 📂 pipeline/              # Orchestration workflows
│   ├── 📁 Injest/          # Data ingestion pipelines
│   ├── 📁 Process/         # Data transformation pipelines
│   ├── 📁 Sqlize/          # Data loading pipelines
│   └── 📁 Execute/         # Orchestration pipelines
├── 📂 trigger/               # Automation schedules
└── 📂 factory/               # Factory configuration
```

## 🔄 Data Pipeline Workflows

### 1. Data Ingestion Pipelines

#### **ECDC Data Ingestion** (`pl_injest_ecdc_file_list`)
- **Trigger**: Daily (24-hour intervals)
- **Function**: Downloads COVID-19 files from ECDC sources
- **Process**:
  1. Lookup file list from configuration
  2. ForEach loop to download multiple files
  3. Store raw data in Data Lake

#### **Population Data Ingestion** (`pl_injest_population_data`)
- **Trigger**: Blob event (when population file arrives)
- **Function**: Validates and processes population data
- **Process**:
  1. Check file exists and validate column count (13 columns)
  2. Copy from compressed (.gz) to TSV format
  3. Delete source file after successful copy

### 2. Data Processing Pipelines

#### **Cases & Deaths Processing** (`pl_processed_cases_deaths`)
- **Dataflow**: `df_transform_cases_deaths`
- **Process**:
  1. Filter data for Europe region only
  2. Select required fields (country, population, indicator, daily_count, date)
  3. Pivot data by indicator (confirmed cases, deaths)
  4. Join with country lookup for standardized codes
  5. Output: `cases_deaths.csv`

#### **Hospital Admissions Processing** (`pl_processed_hospital_admissions_data`)
- **Dataflow**: `df_transform_hospital_admissions`
- **Process**:
  1. Split data into weekly and daily streams
  2. Join weekly data with date dimensions
  3. Pivot by admission type (hospital, ICU)
  4. Sort and transform data
  5. Output: `hospital_admissions_weekly.csv` & `hospital_admissions_daily.csv`

#### **Population Data Processing** (`pl_processed_population_data`)
- **Dataflow**: `df_transform_population_data`
- **Process**:
  1. Parse country code and age group from source
  2. Join with country lookup
  3. Pivot data by age group
  4. Filter out null countries
  5. Output: `population_by_age.csv`

### 3. Data Loading Pipelines (SQL)

#### **Cases & Deaths to SQL** (`pl_sqlize_cases_deaths_data`)
- **Target**: `covid_reporting.cases_and_deaths` table
- **Process**: Truncate and insert processed data

#### **Hospital Admissions to SQL** (`pl_sqlize_hospital_admissions_data`)
- **Target**: `covid_reporting.hospital_admissions_daily` table
- **Process**: Truncate and insert daily admission data

#### **Testing Data to SQL** (`pl_sqlize_testing`)
- **Target**: `covid_reporting.testing` table
- **Process**: Column mapping and data type conversion

## ⚙️ Linked Services

| Service | Type | Purpose |
|---------|------|---------|
| `ls_ablob_covidreportingdl` | Azure Data Lake | Primary data storage |
| `ls_ablob_covidreportingsa` | Azure Blob Storage | Configuration files |
| `ls_sql_covid_db` | Azure SQL Database | Data warehouse |
| `ls_http_cases_deaths_europe_eu` | HTTP Server | ECDC data source |
| `ls_hdi_covid_cluster` | HDInsight | On-demand processing |

## 📅 Automation & Triggers

### Trigger Dependencies:
```
tr_ingest_ecdc_data (Daily)
    ↓
tr_process_cases_and_deaths_data
    ↓
tr_sqlize_cases_and_deaths_data

tr_ingest_ecdc_data (Daily)
    ↓
tr_process_hospital_admissions_data
    ↓
tr_sqlize_hospital_admissions_data

tr_population_data_arrived (Event-based)
    ↓
pl_execute_population_pipeline
```

### Trigger Schedule:
- **Daily Ingestion**: Every 24 hours starting from 2025-05-20
- **Processing**: Triggered after successful ingestion
- **SQL Loading**: Triggered after successful processing
- **Population**: Event-driven when new files arrive

## 🗃️ Data Schema

### SQL Database Tables:

#### `cases_and_deaths`
- country (varchar)
- country_code_2_digit (varchar)
- country_code_3_digit (varchar)
- population (bigint)
- cases_count (bigint)
- deaths_count (bigint)
- reported_date (date)
- source (varchar)

#### `hospital_admissions_daily`
- country (varchar)
- country_code_2_digit (varchar)
- country_code_3_digit (varchar)
- population (bigint)
- reported_date (date)
- hospital_occupancy_count (bigint)
- icu_occupancy_count (bigint)
- source (varchar)

#### `testing`
- country (varchar)
- country_code_2_digit (varchar)
- country_code_3_digit (varchar)
- year_week (varchar)
- week_start_date (date)
- week_end_date (date)
- new_cases (bigint)
- tests_done (bigint)
- population (bigint)
- testing_data_source (varchar)

## 🔧 Configuration

### Git Integration:
- **Publish Branch**: `adf_publish`
- **Git Comments**: Enabled

### Compute Configuration:
- **Core Count**: 8
- **Compute Type**: General
- **Trace Level**: Fine

### Error Handling:
- **Retry Policy**: 30 seconds interval
- **Max Concurrency**: 50
- **Timeout**: 12 hours for copy activities

## 🚀 Getting Started

### Prerequisites:
1. Azure subscription
2. Azure Data Factory instance
3. Azure Data Lake Storage Gen2
4. Azure SQL Database
5. Appropriate permissions for data access

### Deployment:
1. Clone this repository
2. Import ADF artifacts using Azure DevOps or GitHub integration
3. Configure linked services with your Azure resources
4. Update connection strings and credentials
5. Enable triggers to start automated processing

### Manual Execution:
```bash
# Trigger ECDC data ingestion
Trigger: tr_ingest_ecdc_data

# Process specific data type
Pipeline: pl_processed_cases_deaths
Pipeline: pl_processed_hospital_admissions_data
Pipeline: pl_processed_population_data

# Load to SQL
Pipeline: pl_sqlize_cases_deaths_data
Pipeline: pl_sqlize_hospital_admissions_data
```

## 🛠️ Pipeline Creation Steps

### Phase 1: Data Ingestion Using Azure Blob Storage

The population dataset implementation begins with loading data from local storage into Azure Blob Storage as a gzipped file, followed by copying it into Azure Data Lake Gen2 as a TSV file. This process requires creating a storage account and uploading the compressed file into a designated container, then establishing a data lake with a raw container folder for file storage.

The first step involves creating an Azure Data Factory instance and establishing the pipeline structure. Under the manage tab, linked services are created (e.g., `ls_ablob_name`) to establish connections between the storage account, data lake, and ADF for performing copy activities. These linked services serve as the foundation for data movement operations.

A pipeline named `pl_injest_population_data` is created with concurrency set to 1 to ensure sequential processing. The copy data module is utilized for data transfer operations, with general section configurations including retry count and timeout options. Source and sink datasets are connected to the copy activity, followed by validation and debugging of the data copying process.

The validation activity plays a crucial role in real-time scenarios where files are updated daily based on available data. This activity helps determine the optimal execution time and validates file presence before processing. Testing file deletion scenarios demonstrates how the pipeline fails due to timeout when the source file is unavailable.

For the failed case scenario, a timeout feature is set to 30 seconds. Since the file size is manageable, the pipeline fails within this timeframe, demonstrating the importance of proper timeout configuration. The Get Metadata activity is implemented to check file metadata including column count, size, and existence for validation purposes.

An If condition activity is implemented to check for potential failures and validate metadata before performing the copy operation. Using dynamic content, the condition checks if the column count from the Get Metadata activity matches the expected value (13 columns) before proceeding with the copy behavior. The True condition validates the copy behavior, while the False condition triggers a failure case.

The Fail activity is implemented to test its purpose, with the condition in the If activity adjusted accordingly. For the False case, the Fail activity is utilized to handle error scenarios. A Delete activity is implemented immediately after the copy activity execution to remove the source file, with logging enabled in the Delete Activity settings for audit purposes.

An event-based trigger is implemented for the pipeline, automatically starting execution upon source file insertion. After deletion, logging is visible as it's enabled in the Delete Activity settings. The Storage Events Trigger is implemented as a more suitable solution, automatically activating when files arrive in the storage account and transforming them through subsequent pipeline steps.

### Phase 2: Data Ingestion from HTTP Sources

The HTTP ingestion process involves multiple ECDC data sources including cases and deaths, country response, hospital admissions, and testing data from GitHub repositories. The implementation follows a systematic approach of creating linked services, source and sink datasets, and pipelines for data copying operations.

To handle multiple datasets efficiently, parameters and variables are implemented in the pipeline to import multiple datasets simultaneously by passing URLs as parameters into linked services. Dataset parameters are created for relative URLs and filenames, while pipeline variables are initialized and declared with appropriate values.

The pipeline is debugged specifically for hospital_admissions.csv to ensure proper functionality. The cases and deaths datasets and pipeline are generalized for ECDC use, leveraging the parameterized approach for any dataset type. A schedule trigger is created and parameters are passed from the trigger into the pipeline to retrieve hospital admissions data.

Control flow activities are implemented to handle multiple file imports efficiently. Lookup and ForEach activities are utilized for retrieving URLs from JSON documents, enabling dynamic processing of multiple data sources. The baseURL is parameterized throughout the entire pipeline chain, from linked services through datasets, pipelines, and copy activities.

The trigger is configured to use Lookup and ForEach activities, eliminating the need for pipeline parameters. The copy activity parameters for sink and source are changed to use ForEach activity output and input, with the ForEach activity iterating over values from the Lookup activity. URLs are structured as JSON objects and passed to the pipeline using the Lookup activity, with each value iterated using the ForEach activity and passed to the copy ECDC Data activity.

### Phase 3: Data Transformation Implementation

Data transformation begins with creating a Dataflow for the cases and deaths dataset. The transformation process includes source, filter, select, pivot, lookup, and sink transformations, culminating in pipeline creation for execution. Dataflows require source and sink transformations as mandatory components, with the flexibility to include multiple intermediate transformations as needed.

Microsoft recommends maintaining a minimum number of transformations for optimal performance. These transformations can be tested on debug clusters, with the transformation logic converted to Spark and executed on Microsoft Databricks clusters for distributed execution.

For the cases and deaths dataflow, the data is examined to identify that while all countries' data is available, only European data is utilized for this implementation. The country code structure varies between 2-digit and 3-digit formats across different files. Instead of maintaining separate indicators and daily counts, the data is standardized to maintain confirmed cases count and deaths count in standard rows. The date column is replaced with reported date, and the rate_14_days field is removed.

Country code transformation is implemented using a lookup file. The big cases deaths dataset is used for validation purposes during debugging, with country lookup data containing information for GBR (UK) and India for validation testing.

Source and sink transformations are mandatory components in every dataflow. Sources can be linked to datasets or implemented inline, with dataset linking being preferred for multiple dataflow usage while inline sources have limitations and limited support. Source settings include options for allowing or limiting schema drift, which handles changes in the schema structure.

Sampling in dataflows typically picks few samples from the source for validation. In this implementation, validation is performed manually in debug settings, focusing on country-based cases and deaths for UK and India. Source options include post-transformation source deletion or moving transformations to new files.

Projection automatically detects column names and types using internal inference mechanisms. The inspect and preview features show the exact output after transformation, enabling developers to verify the transformation results before execution.

### Phase 4: Advanced Data Transformation Features

Debug mode is initiated for the first data transformation of the cases and deaths file. The source step utilizes named datasets with the choice between dataset or inline options. Choosing dataset provides advantages for multiple use cases, while inline sources have very limited support. Schema options and sampling data options are configured, with schema validation ensuring new columns are not accepted and sampling set to manual for controlled data selection.

Projection automatically detects column data types, while the inspect tab displays the output schema after transformation and preview shows sample data. The Filter Activity uses the visual editor for necessary manipulations where all required transformations can be performed efficiently.

Select Transformation focuses on necessary columns with rule-based mapping and mapping options. Rule-based mapping leverages visual studio capabilities for performing necessary transformations, with options for removing duplicate columns. In this activity, continent and rate_14_days are deleted, and reported date is renamed using rule-based mapping.

Pivot Transformation is implemented for pivoting indicator columns and daily counts into confirmed cases count and deaths count. Non-participating columns fall under the group by section, with pivot keys representing the converting columns. The pivot columns can be navigated using the visual code editor for writing custom transformation code.

Lookup Transformation is used to include country codes of both 2-digit and 3-digit formats. A source dataset is created for country lookup, and the lookup transformation is implemented with necessary changes and conditions for combining the data sources.

A sink is created and a container is added to store all processed or transformed data. Select Transformation is used to add or remove fields from the lookup transformation after data flow completion.

Conditional split is required for data that contains both weekly and daily information. The data can be divided based on daily and weekly patterns, with weekly data having null date values. Start and end dates of the week can be redefined using a custom dimdate lookup file.

For the dimdate source transformation, derived transformation is used to modify or create new columns where exact matching creates necessary columns in original existing columns with mapping to the dimdate column.

### Phase 5: HDInsight and Azure Databricks Integration

After the transformation step, input data preparation is required for running on HDInsight Activity. This process is similar to Spark, where Spark runs on distributed clusters and HDInsight works in the same manner. These systems expect input data as folders and divide data into chunks for processing. While Data Factory can work directly with files or datasets, HDInsight requires input as folders for optimal performance.

The cluster configuration uses a single node setup with node type Standard_D4a_v4. Azure Databricks Service is created using the available subscription, which navigates to the Databricks workspace upon creation. The workspace environment uses Azure ID for login authentication.

Azure Databricks clusters are created with computation resources consisting of one driver node and one or more worker nodes. Two cluster types are available: All-purpose/Interactive clusters and Job Clusters, with the latter being automatically terminated and non-restartable once terminated.

For cluster creation, the process involves selecting "New → Create Cluster → Give Name → Choose Single node (for free tier) → Performance with LTS (Long term support) → Set timer for default cluster deletion → Choose node type with cost considerations."

To use data lake in Azure Databricks, an Azure service principal is created and granted access to the data lake. Mounts are created in Databricks using the service principal for seamless data access.

Azure Service Principal is created, attached to the data lake, and tokens are generated for data lake attachment to Azure Databricks for mounting purposes. The mounting Python script is executed to keep containers that need processing on Databricks.

In ADF, a pipeline is created to run the Databricks cluster by adding the custom cluster. Population transformation focuses on 2019 population data, filtering out other years. The dataset consists of age group and country code information that needs to be split and processed accordingly.

### Phase 6: SQL Database Integration and Data Loading

After transformation, all data is copied to SQL databases. A SQL database is created and Azure Data Studio is used to access the database. SQL Server is connected to Azure Data Studio for database management.

To copy data from data lake to SQL, the pipeline creation facility is utilized with copy data activity. The source is set as the processed dataset and the sink as the SQL table, requiring creation of linked services and datasets for the sink as well. The dataset is integrated with SQL and test connection is performed using the linked service. Linked services act as variables declared in a class for all datasets.

For source configuration, the name is provided along with the created dataset, and wildcard path is used to automatically pick cases deaths, hospital admissions, or testing data. In the sink, a pre-copy script is written as Truncate to prevent data duplication when the pipeline runs multiple times. For testing, mapping is used to map source columns, with unmapped columns in the dataset being mapped accordingly.

### Phase 7: Data Orchestration and Pipeline Dependencies

Data orchestration requirements include full automation of pipeline executions, regular interval or event-based pipeline runs, and activities running only after upstream dependencies are satisfied. This enables easier monitoring of execution progress and issue identification.

Capabilities include dependencies between activities within a pipeline, dependencies between pipelines within a parent pipeline, dependencies between triggers (specifically Tumbling window triggers), and custom-made solutions for complex orchestration scenarios.

For population data, a pipeline is created to run population processing. Population ingestion creates an ingest pipeline that automatically triggers using storage event triggers when files arrive with specified starting names. This pipeline is connected to the processing pipeline with custom Databricks cluster integration. Upon file arrival, the first pipeline executes, followed by Databricks cluster triggering, creating complete pipeline orchestration for population data.

Processed population data is only invoked upon completion of the ingest population data. Event triggers are used to invoke the ingest population data pipeline. In the actual plan, the processing pipeline is executed on Databricks cluster, but due to free tier limitations, dataflow is used as a built-in Azure Data Factory alternative.

### Phase 8: Trigger-Based Orchestration and Dependency Management

Trigger-based orchestration is implemented where all triggers are created for each pipeline as tumbling window triggers. Dependencies are added sequentially, creating a chain of execution. Tumbling window triggers are created for five pipelines: cases and deaths, hospital admissions pipelines, with the first pipeline triggering ECDC file list, second pipeline triggering processed cases deaths and hospital admissions, and third pipeline triggering SQLize cases deaths and hospital admissions.

Advanced settings include offset and window size configurations for scenarios where yesterday's data is processed today and today's data is processed tomorrow. This provides flexibility in data processing schedules and timing.

### Phase 9: Monitoring and Performance Management

Monitoring capabilities in Azure Data Factory include resource monitoring (size, total objects), integration runtime monitoring (CPU, available memory), trigger runs monitoring (success/fail, issue resolution), pipeline runs, and activity runs tracking.

The Data Factory Monitor provides the ability to monitor pipeline/trigger status, rerun failed pipelines/triggers, send alerts based on metrics, access base level metrics and logs, with pipeline runs stored for 45 days.

Azure Monitor offers the ability to route diagnostic data to other storage solutions, provides richer diagnostic data, enables complex queries and custom reporting, and supports reporting across multiple data factories.

In the monitor tab, email or SMS alerts can be created using the Alert and Metrics tab in ADF Monitor. Reporting is available through the Metrics tab under the Alert and Metrics section. New alert metrics can be created with names, severity levels, target criteria, time gaps for failure notifications, and configuration for email or mobile notifications.

Pipeline rerun options include complete pipeline rerun or rerun from the point of failure. For dependent triggers like tumbling window triggers, running the complete trigger is recommended to ensure proper dependency resolution and successful execution.

Metrics under Alert and Metrics tabs show all available metrics with filtering capabilities. Failed operations can be selected and investigated. In the monitoring tab, metrics can be pinned and saved to dashboards for continuous monitoring.

Azure Monitor diagnostic settings allow capturing all metrics for specific resources and storing them in designated locations. Retention periods can be configured for specific day outcomes.

### Phase 10: Log Analytics and Kusto Query Language

For creating log analytics workspace, the process involves naming the workspace, choosing resource group, selecting pricing tier, reviewing and creating the workspace. After creation, diagnostic settings are searched in ADF and log analytics workspace is added where logs will be analyzed. Destination tables are set to resource-specific configurations.

Log Analytics provides queries and extensive information about log analytics. For querying in Log Analytics workspace, Kusto Query Language (KQL) is used, which is similar to SQL but with some differences. KQL becomes essential when dealing with large numbers of pipelines that require querying.

After query execution, graphs can be generated and added to dashboards. Kusto Query Language includes options for generating charts based on query execution, with bar chart code implementation possible within the queries themselves.

### Phase 11: Power BI Integration and DevOps Implementation

Power BI Desktop introduction provides capabilities for data visualization and business intelligence reporting, enabling stakeholders to create interactive dashboards and reports from the processed COVID-19 data.

Continuous Integration/Continuous Delivery (CI/CD) implementation follows DevOps characteristics including collaboration, trust and transparency, agile development approach, continuous integration/delivery, automation and continuous improvement.

The project lifecycle follows a continuous loop: Plan → Code → Build → Test → Release → Deploy → Improve → Monitor → Plan, with continuous integration, continuous delivery/deployment, and continuous improvement phases.

For Java projects, .class files are generated, while in ADF, JSON and ARM Template files are produced. ARM Template testing can be performed using .NET code for validation purposes.

### Phase 12: Manual Integration and Automated Delivery

Manual integration with automated delivery is implemented in ADF. Previously, ADF publish and release scripts in deployments were used, but Microsoft now provides a package for deployment that automates the build in deployment pipelines.

Azure DevOps consists of Boards, Repos, Pipelines, Test Plans, and Artifacts. The organization structure can contain one or more organizations for companies based on business units, with each organization potentially containing multiple projects.

New Data Factories are created with naming format `dev-ci-cd-demo-adf-vchalla3` for Dev, Test, and Prod environments for DevOps purposes. In the Repos section of Azure DevOps, git branches are created with recognizable synonym names.

Policies are set for main branches to prevent direct access. Feature branches are created for pipeline development and debugging, followed by pull request creation, review, approval, and code movement to the main branch.

### Phase 13: Pipeline Development and Git Integration

A simple demo pipeline is created with Wait activity for demonstration purposes. The pipeline can be validated and debugged, but publishing is restricted to feature branches where only commit changes are allowed. Once committed, the JSON code of the pipeline becomes visible in Azure Repo.

After committing changes, pull requests are created from ADF in the working directory. In Azure DevOps, pull requests become visible for review and approval. Typically, team leads or designated personnel review and approve code changes.

After approval, the merge can be completed, making changes visible in the main branch. Every time a branch is created and saved, changes are committed to the branch. To update the branch, the same procedure is repeated: create new branch → save changes → create pull request → review and approve → complete merge.

The repository can be published by clicking publish in the main branch, creating an ADF publish branch where all repository changes are stored under ARM Templates. Before publishing, live mode is checked to ensure no changes or pipelines are visible, and after publishing, changes become visible in Live mode with the created pipeline as the publish generates ARM Templates.

After publishing in the repo, the ADF publish branch becomes visible with ARM templates and necessary dependency folders. Tasks are added and deployment template types are selected using ARM template for the pipeline. Settings are edited to create Azure Service Principal and select the resource group (e.g., test-ci-cd-demo-vchalla3-rg).

Linked artifacts and branches are selected, and variables are created for pipeline reuse. After creating the artifact template, manual triggers can be executed (Create release) to run the pipeline. After the first manual deployment, continuous deployment is chosen, enabling automatic deployment when changes occur in the branch from feature branches merged to main branch and published.

### Phase 14: Advanced Deployment and Error Handling

When objects are deleted or active triggers are updated, errors may occur. For example, in release_1, the second pipeline is deleted and a trigger is created for the second release. A tumbling window trigger is created and saved, followed by pull request creation and Azure DevOps approval. The pipeline including the trigger and pipeline deletion is approved.

Object deletion doesn't occur in test environments because ARM templates only include objects without deletion capabilities. The created trigger needs to be started and deployed. When changes are made to the trigger, they are automatically saved, but publishing may fail in the deployment process if active triggers cannot be updated.

To fix deployment process issues, pre-deployment and post-deployment scripts are required. Microsoft provides scripts in GitHub repositories with two types: one that stops all active triggers regardless of changes, and another that stops only the specific trigger that needs changes. Microsoft provides comprehensive documentation with commands for running these scripts.

### Phase 15: PowerShell Integration and Production Deployment

A feature branch is created with a new folder (release folder) for PowerShell scripts. Files are committed/uploaded to the folder, followed by pull request creation and merge to main. The file is added as an artifact to the pipeline, and PowerShell template is added with necessary changes for activation.

Script path and arguments (commands) are provided with pre and post deployment arguments. Pre and post deployments are configured based on requirements, and two templates are created accordingly. Changes are merged to the main branch.

The next stage involves adding the production pipeline to the actual pipeline and removing static values to create pipeline variables. Hardcoded values are kept in ARM templates for resource group names, deployment locations, and data factory names. Variables are created for all these values to enable pipeline extension to production stages.

When selecting pipeline environment, the variables section allows choosing between pipeline variables or variable groups. For this project, only pipeline variables are used for the three discussed components. Variables are added with scope fixed based on test or production environments.

Changes are made using the `$()` syntax where actual values are present to make the pipeline independent of variables. These changes are implemented in pre-deployment and post-deployment scripts. Manual release is created instead of auto-deployment, and the current pipeline is selected to add a new stage with production environment variables.

The test stage is cloned (not copied) with only variable changes for the production environment. Service principal access is granted for production deployment. When cloning the test stage, pre-deployment conditions for production are included to ensure approval before deployment.

For testing purposes, a test pipeline is created where changes can be committed. Production doesn't participate in continuous deployment as it requires approval from designated personnel. After releases, pipeline deployment status can be verified. Pipeline variables are used in this project, but variable groups offer more insights about data.

### Phase 16: Automated Build and Deployment

In option 1 implementation, complete automation isn't possible, but automation is attempted in option 2. After approving pull requests, manual publishing is often required to generate ARM templates and publish changes from main branch to successive branches. The publish and ARM template generation can be automated after successful pipeline builds using npm packages, with Microsoft providing necessary documentation for implementation.

For automated build pipeline implementation, it's good practice to create feature branches and push changes to main branches. For build-related support files, a feature branch is created with a build folder containing a README file where all files will be included later. Microsoft provides scripts to automate pipeline deployment using successful builds, with code provided in YAML files.

A new pipeline is created, cloned, and renamed from the first one. The artifact from the repository is deleted and replaced with the new build pipeline created using the YAML file. The ARM Template location is changed from the repository to the ARM template generated by the build pipeline.

In the new pipeline, ARM Template comes from the build pipeline rather than the ADF publish branch, requiring artifact deletion and creation of a new one pointing to the build pipeline. In the stage, ARM Template pointing needs to be changed by going to tasks and changing ARM Template. The ARM Template path is copied and changed in pre and post deployment templates as well.

Changes are made for test and production environments. A dev stage is created from test clone, and stage order is changed to dev → test → prod to avoid errors. Before deployment, service principal access is granted for the resource group to prevent errors. All stages are deployed and verified.

### Phase 17: Environment-Specific Configuration and Final Deployment

Before implementing continuous deployment, manual release is used to test functionality. The artifact option is changed to deploy after build completion. The pipeline is run and verified for proper operation.

When working with release pipeline scenarios, dev environment resources (data lake, SQL server) are used from the dev environment. For test and production environments, these resources need to be changed to point to their respective environments. Access permissions need to be provided, and resources need to be created and attached to artifacts.

For data lakes or other resources being created, naming conventions should match the data factory naming structure for consistency and maintainability.

## 📈 Monitoring & Maintenance

### Key Metrics to Monitor:
- Pipeline success rates
- Data freshness
- Processing duration
- Error rates
- Resource utilization

### Maintenance Tasks:
- Regular trigger monitoring
- Data quality validation
- Performance optimization
- Cost management
- Security updates

## 🏷️ Tags & Metadata

- **Domain**: Healthcare Analytics
- **Data Source**: ECDC, European Statistics
- **Update Frequency**: Daily
- **Data Retention**: Configurable
- **Compliance**: GDPR compliant

## 📞 Support

For issues and questions:
1. Check Azure Data Factory monitoring
2. Review pipeline run history
3. Check trigger execution logs
4. Validate data source availability

---

**Last Updated**: May 2025
**Version**: 1.0
**Environment**: Production Ready
