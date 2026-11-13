# FoodFleet Analytics — End-to-End Data Engineering Project

## 1. Project Overview

FoodFleet Analytics is an end-to-end data engineering and business intelligence project built using Microsoft Fabric.

The project transforms operational data from an on-premises SQL Server database into analytics-ready data using the Medallion Architecture (Bronze, Silver and Gold).

The final Gold Warehouse serves a Power BI dashboard that provides business insights into revenue, orders, customers, restaurants, payments and delivery performance.

*Key objectives:*
- Build a metadata-driven data ingestion framework.
- Implement incremental data loading using watermarks.
- Develop Bronze-to-Silver transformations using Fabric notebooks.
- Implement Slowly Changing Dimension Type 2 (SCD2) for customer history.
- Build a dimensional data model in a Fabric Warehouse.
- Orchestrate pipelines using a master pipeline.
- Implement execution auditing and validation.
- Develop a Power BI analytics dashboard.

## 2. Technology Stack

| Technology | Purpose |
|---|---|
| Microsoft Fabric | Unified analytics platform |
| Fabric Data Factory | Pipeline orchestration and data movement |
| Fabric Lakehouse | Bronze and Silver storage |
| Fabric Notebooks | PySpark transformations |
| Fabric Warehouse | Gold dimensional model |
| T-SQL | Stored procedures and SQL transformations |
| Python / PySpark | Data processing |
| Power BI Desktop | Dashboard development |
| Power BI Semantic Model | Business measures and relationships |
| On-premises Data Gateway | Connectivity to local SQL Server |

## 3. Architecture

The project follows the Medallion Architecture.

### End-to-End Data Flow

text
Local SQL Server
FoodFleetSourceDB
       |
       v
On-premises Data Gateway
       |
       v
Bronze Ingestion Pipeline
       |
       v
Bronze Lakehouse
lh_foodfleet_bronze
       |
       v
Bronze-to-Silver Pipeline
       |
       +-------------------------+
       |                         |
       v                         v
Silver Processing          SCD Type 2
Notebook                   Notebook
       |                         |
       v                         v
Silver Lakehouse
lh_foodfleet_silver
       |
       v
Gold Processing Pipeline
       |
       v
Gold Stored Procedures
       |
       v
Gold Warehouse
wh_foodfleet_gold
       |
       v
Power BI Semantic Model
       |
       v
FoodFleet Analytics Dashboard


A master orchestration pipeline named pl_foodfleet_daily_orchestration coordinates the three main pipelines sequentially.

## 4. Source Data

Source database: FoodFleetSourceDB

The project uses six operational tables:

| Source Table | Description |
|---|---|
| Customers | Customer information |
| Restaurants | Restaurant details |
| Orders | Customer order information |
| OrderItems | Individual items within orders |
| Delivery | Delivery and delivery partner details |
| Payments | Payment transactions |

The source tables contain ModifiedDateTime and IsDeleted columns to support incremental processing and soft-delete handling.

## 5. Bronze Layer — Data Ingestion

*Lakehouse:* lh_foodfleet_bronze

The Bronze layer stores ingested source data.

### Implementation

- Created a metadata-driven ingestion framework.
- Used Fabric Data Factory Copy activities to extract data from SQL Server.
- Connected the local SQL Server through an On-premises Data Gateway.
- Implemented incremental extraction using ModifiedDateTime.
- Used a watermark table to track the last successful extraction point.
- Configured sequential execution using a ForEach activity.
- Updated watermarks after successful data movement.

### Metadata Tables

| Table | Purpose |
|---|---|
| dbo.IngestionConfig | Stores source and destination configuration |
| dbo.IngestionWatermark | Stores incremental extraction watermarks |
| dbo.IngestionAudit | Stores ingestion audit information |

## 6. Silver Layer — Data Transformation

*Lakehouse:* lh_foodfleet_silver

Notebook: nb_foodfleet_silver_processing_v2

The Silver layer contains cleaned and standardized data.

### Transformations

- Read Bronze tables dynamically using configuration metadata.
- Selected required columns.
- Trimmed string columns.
- Removed invalid business keys.
- Filtered soft-deleted records.
- Removed duplicate business keys.
- Retained the latest record based on modification timestamp.
- Wrote transformed data as Delta tables in the Silver Lakehouse.

### Silver Configuration

The dbo.SilverConfig table controls source tables, target tables and active processing configurations.

### Silver Tables

- silver_customers
- silver_restaurants
- silver_orders
- silver_orderitems
- silver_delivery
- silver_payments

## 7. Slowly Changing Dimension Type 2

Notebook: nb_foodfleet_scd2_processing

Implemented SCD Type 2 to maintain historical customer information.

### Implementation

- Used a metadata-driven SCD configuration table.
- Identified customers using CustomerID.
- Compared customer attributes across historical records.
- Preserved previous customer versions.
- Created new versions when tracked attributes changed.
- Maintained current and historical records.
- Included an IsDeleted column in the history table.

*Example:*

| CustomerID | CustomerName | City | Record Status |
|---|---|---|---|
| 1 | Ravi Kumar | Mysuru | Historical |
| 1 | Ravi Kumar | Bengaluru | Current |

This allows historical customer information to remain available for analytical reporting.

Target table: silver_customers_history

## 8. Gold Layer — Dimensional Data Warehouse

*Warehouse:* wh_foodfleet_gold

The Gold layer contains the dimensional model used for business intelligence.

### Dimension Tables

- dbo.DimCustomer
- dbo.DimDate
- dbo.DimRestaurant
- dbo.DimDeliveryPartner

### Fact Tables

- dbo.FactOrders
- dbo.FactOrderItems
- dbo.FactPayments
- dbo.FactDelivery

### Gold Processing

Gold data loading is implemented through T-SQL stored procedures.

The master procedure is:

dbo.usp_RunGoldPipeline

It executes the following procedures:

1. usp_LoadDimDate
2. usp_LoadDimRestaurant
3. usp_LoadDimDeliveryPartner
4. usp_LoadDimCustomer
5. usp_LoadFactOrders
6. usp_LoadFactOrderItems
7. usp_LoadFactPayments
8. usp_LoadFactDelivery
9. usp_ValidateGold

The dimensional model supports analysis across customers, restaurants, dates, orders, payments and deliveries.

## 9. Pipeline Orchestration

Master pipeline: pl_foodfleet_daily_orchestration

The master pipeline executes three child pipelines sequentially.

| Sequence | Pipeline | Responsibility |
|---|---|---|
| 1 | pl_foodfleet_ingestion | SQL Server to Bronze |
| 2 | pl_foodfleet_bronze_to_silver | Bronze to Silver and SCD2 |
| 3 | pl_foodfleet_gold_processing | Silver to Gold |

Pipeline dependencies are configured using success and failure conditions.

The master pipeline is configured with a daily 9:00 AM IST schedule.

## 10. Auditing and Monitoring

Auditing is implemented at multiple processing stages.

| Audit Table | Responsibility |
|---|---|
| dbo.BronzeToSilverAudit | Tracks Silver notebook execution |
| dbo.GoldPipelineAudit | Tracks Gold stored procedure execution |
| dbo.DailyOrchestrationAudit | Tracks master pipeline execution |

Audit information includes relevant execution identifiers, timestamps, processing status, row counts and error information.

The Gold and orchestration stored procedures update execution status and capture errors.

## 11. Power BI Dashboard

The Power BI report is built using the Gold Warehouse semantic model.

### Business KPIs

- Total Revenue
- Total Orders
- Average Order Value
- Total Customers
- Total Items Sold
- Average Delivery Time

### Analytical Visuals

- Revenue by Restaurant
- Monthly Revenue Trend
- Order Status Distribution
- Average Delivery Time by Partner
- Revenue by Payment Method

The dashboard provides an executive-level overview of FoodFleet's business performance.

## 12. Key Engineering Concepts Demonstrated

- Medallion Architecture
- Metadata-driven ingestion
- Incremental data loading
- Watermark management
- PySpark DataFrame transformations
- Delta Lake tables
- Deduplication
- Soft-delete handling
- Slowly Changing Dimension Type 2
- Star Schema dimensional modelling
- Fact and dimension loading
- T-SQL stored procedures
- Pipeline orchestration
- Execution auditing
- Data quality validation
- Power BI semantic modelling and DAX

## 13. Future Improvements

Potential enhancements for a production deployment:

- Complete automated ingestion audit integration.
- Implement true incremental SCD Type 2 processing.
- Improve CDC-based change tracking.
- Add automated pipeline retry and alerting.
- Implement comprehensive data quality checks.
- Add CI/CD deployment across development, test and production workspaces.
- Implement role-based security and sensitivity controls.
- Optimize large-scale data processing and warehouse queries.

## 14. Conclusion

FoodFleet Analytics demonstrates an end-to-end Microsoft Fabric data engineering solution, covering data ingestion, transformation, historical data management, dimensional modelling, orchestration, auditing and business intelligence.

The project demonstrates practical experience with SQL Server, Microsoft Fabric, PySpark, Fabric Data Factory, Fabric Lakehouse, Fabric Warehouse and Power BI.
