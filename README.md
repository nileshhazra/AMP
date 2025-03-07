# **Incremental Data Migration Project - AMP (Account Management Programme)**

## **1. Project Overview**
### **Objective:**
The goal of the AMP (Account Management Programme) is to ensure that policy, cover, member details, and adjustments from the **UTOM application (Underwriters Target Operating Model)**—which feeds data into **SPLTOMDB**—are synchronized with **Microsoft Dynamics 365**. The data in Dynamics CRM is used for reporting and operational purposes, including reports for **Medisea (a medical-related service).**

### **Business Context:**
- This project is for **Shipowners Pvt Limited (SPL)**, a **UK-based marine insurance company**.
- The data primarily relates to **ship insurance**, where ships are referred to as **vessels or risks** in the database.
- Data is migrated from **two source databases** to **Microsoft Dynamics 365**.

### **Technology Stack:**
- **Source Databases:**
  - `SPLTOMDB` (SQL Server) → Database: **TOM**
  - `SIGMADB02` (SQL Server) → Database: **Sigma** (Legacy System)
- **Landing/Staging Layer:**
  - Server: **SPLDW**
  - Database: **DATA_AMP_MIGRATIONDB**
- **Target:**
  - **Microsoft Dynamics 365** (Entities: Opportunity, Quote, QuoteDetail, SalesOrder, SalesOrderDetail, Product, etc.)
- **ETL Tool:** Talend Studio
- **Execution:**
  - Talend-generated build files are copied to servers and executed via **Windows Task Scheduler** (`.bat` file) **nightly**.

---

## **2. Source Database Details**
### **SPLTOMDB (TOM Database - New System)**
- The **primary system** where all new policies are created.
- **Key Tables:**
  - **Policy & Risk Data:** ApplicationBase, Risk, Risk_Vessel, RiskNonvessel, ref_RiskStatus, ref_vesseltype, ref_vesselcategory
  - **Member & Assured Data:** MemberAssured, ApplicationMemberAssured, memberassuredgroup, Party
  - **Policy & Cover Data:** ApplicationPolicyStatus, ref_PolicyStatus, ApplicationCover, TemplateCover, Cover, CoverCategory
  - **Broker & Other References:** ApplicationBroker, ApplicationType, ref_country, Branch
  - **Premium & Transactions:** Premium, ref_currency, JournalEntry, Transaction, transactionpremium, ref_premiumratingbasis
  - **Crew Data:** riskPreDeliveryCrew, riskPostDeliveryCrew

### **SIGMADB02 (Sigma Database - Legacy System)**
- **Legacy system**, no new data is created here, but it remains part of migration jobs.
- **Key Tables:**
  - **Policy & Risk Data:** app_policy, app_risk, app_vessel, ref_vesseltype, ref_vesselcategory, ref_geographicarea
  - **Member & Party Data:** app_party
  - **Policy Terms & Premium Data:** app_policyterm, app_premium, ref_premiumtype
  - **Branch & Office Data:** app_branch, app_office
  - **MTA (Mid-Term Adjustments) Data:** app_mtacoverchange, app_mtavesselchange

---

## **3. Subjobs in AMP Migration Process**

### **Subjob 1: TomMigrationDisableJob**
#### **Purpose:**
- Disables a specific job in **TOM database** before migration begins.
#### **Execution Code:**
```sql
DECLARE @JobStatus INT= 0,
@isEnabled INT= 0;

EXEC dbo.ChangeRestoreJobStatus
@isEnabled,
@JobStatus OUT;

SELECT @JobStatus
```
#### **Explanation:**
- Calls the stored procedure `ChangeRestoreJobStatus` to disable a restoration job before migration starts.
- Ensures no conflicts between migration and background processes.

### **Subjob 2: Incremental_MasterJob**
#### **Purpose:**
Migrates **master data** from source databases to **staging (DATA_AMP_MIGRATIONDB)** and then to **Dynamics 365**.

#### **Child Jobs:**
1. **Incremental_MasterDataJob_Staging**
   - Migrates data from **TOM** tables: `cover_category`, `cover`, `ref_vesseltype`, `ref_vesselcategory`, `risk_subtype`, `ref_riskstatus`, `ref_risktype`, `riskvessel`, `risknonvessel`, `riskpredeliverycrew`, `riskpostdeliverycrew`, `Risk`.
   - Inserts data into staging tables: `Cover`, `Cover_category`, `VesselType`, `VesselCategory`, `Risk`.

2. **Incremental_MasterDataJob_CRM**
   - Moves data from **staging** to **Microsoft Dynamics CRM**:
     - **Entities:** `spl_riskcategories`, `spl_vesseltypes`, `spl_risks`, `Products (cover_category data)`, `Products (Cover data)`, `Accounts (membergroup data)`, `Accounts (membergroup master data)`.

3. **MemberRenewalDate**
   - Syncs **MemberAssured** data from `TOM` to `Accounts (spl_memberrenewaldate column)` in CRM.

### **Subjob 3: Incremental_UTOM_Quotes_Job**
#### **Purpose:**
- Migrates **quote data** from `TOM` to staging for further processing.
- Handles **quotes that have not yet been converted to policies**.

#### **Process:**
1. Inserts quote data into **TomQuote_L** and **Quote_Staging** tables in **DATA_AMP_MIGRATIONDB**.
2. Updates `Migration_Status` in **TomQuote_L** from `'NotMigrated_I'` / `'NotMigrated_U'` to `'Migrated_To_Staging'`.
3. Filters relevant quote records using:
   ```sql
   WHERE Discriminator IN ('SigmaRenewalQuote', 'AdditionalCoverQuote', 'RenewalQuote', 'Quote')
   ```

### **Subjob 4: Incremental_Policies_Landing**
#### **Purpose:**
Migrates **policy-related data** from **Sigma (legacy) and TOM (new system)** to staging for further processing.

#### **Child Jobs:**
1. **SigmaBusinessType**
   - Extracts **business type data** from **Sigma** and inserts it into `Sigma_BusinessType` table in `DATA_AMP_MIGRATIONDB`.

2. **Incremental_SigmaPolicies** _(High-Level Guess)_
   - Extracts policies from **Sigma (legacy system)**.
   - Likely filters for relevant policies based on `timestamp` columns.
   - Inserts data into **staging tables** for further processing.

3. **Incremental_TomPolicies** _(High-Level Guess)_
   - Extracts **new and updated policies** from **TOM (new system)**.
   - Likely applies incremental logic using timestamps or status flags.
   - Inserts data into **staging tables** for further migration.

---

## **Next Steps**
- Detailed breakdown of **Incremental_SigmaPolicies** and **Incremental_TomPolicies**.
- Mapping **staging to CRM** transformations.
- Understanding **incremental processing logic** (timestamp-based filtering, status updates, etc.).

---
This document will be updated as we progress further into the AMP migration project.

