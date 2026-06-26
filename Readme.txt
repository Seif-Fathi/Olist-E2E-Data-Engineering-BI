# 🇧🇷 Olist E-Commerce End-to-End BI & Data Engineering Project

An enterprise-grade Data Engineering and Business Intelligence project utilizing **Python**, **SQL Server (T-SQL)**, and **Power BI** to clean, model, and analyze over 100k Brazilian e-commerce orders from the Olist dataset.

[Comprehensive Metadata & Schema Catalog](./Olist_documentation.md)



### Interactive Dashboard Walkthrough (2x Speed Demo)

To demonstrate the sub-second calculation speeds, dynamic tooltips, and seamless enterprise navigation patterns engineered into this report, a high-density video demo is available. 

Click the link below to watch the live interactions, segment-level drill-throughs, and responsive UX layouts:

**[Watch the Interactive Dashboard Demo Video](./power_bi/Olist_BI_Dashboard_2x_Demo.mp4)**

*Note: If GitHub does not render the MP4 directly in your browser, you can clone the repository or view it inside the `./power_bi/` directory.*

---

## Tech Stack
* **Data Pipelines & Ingestion:** Python (Pandas, SQLAlchemy, PyODBC)
* **Database Architecture & Data Modeling:** Microsoft SQL Server (SSMS)
* **Data Warehouse Schema:** Star Schema (Fact & Dimension Tables)
* **Reporting & Analytics:** Power BI *(Phase 3)*

---

## Project Architecture & Workflow

[Production Python Notebook](./notebooks/olist E2E pipline.ipynb)

### Phase 1: Data Cleaning & Initial Handling (Python)
Inspected 9 relational datasets, resolved missing values ($NaNs$), handled duplicate entries, and parsed corrupted datetime fields. Instead of blindly dropping rows or using standard mathematical imputations (like Mean/Median), data-driven business decisions were applied based on **Relational Integrity** and **Business Analytics principles**:

#### 1. Products Table (`product_category_name`, dimensions, and weights)
* **The Issue:** There are 610 products with missing category names, descriptions, and physical dimensions ($NaN$). 
* **The Reality:** These `product_id`s are linked to actual historic transactions in the `order_items` table. Dropping them would lead to financial discrepancies (under-reporting total revenue and net sales).
* **The Decision:** * Filled `product_category_name` with **`'unknown'`** to retain the revenue data while keeping the product visibly unclassified in the dashboard.
  * Filled physical dimensions and weights with **`0`** instead of the Mean/Median. Imputing global averages in a highly diverse e-commerce marketplace (ranging from 50g keychains to 40kg furniture) would heavily distort shipping metrics, inflate logistics aggregations, and mislead logistics performance tracking in Power BI.

#### 2. Reviews Table (`review_comment_title` & `review_comment_message`)
* **The Issue:** Tens of thousands of rows contain missing text comments.
* **The Reality:** This is standard customer behavior; many buyers prefer to leave a quick star rating (1-5) without spending time writing a textual review.
* **The Decision:** Filled missing titles and text with **`'No Title'`** and **`'No Message'`**. This ensures that text-mining or NLP analytics can be safely performed on valid data strings later, without causing exceptions or runtime errors in the pipeline.

#### 3. Orders Table (Missing timestamps like `order_delivered_customer_date`)
* **The Issue:** Missing dates in order fulfillment milestones.
* **The Reality:** These represent open, processing, or canceled orders that never reached the final delivery stage.
* **The Decision:** **Kept as `NULL`**. Injecting fake or average dates here would completely ruin the accuracy of calculating **Delivery Lead Times** and **Logistics Delays** during the Data Modeling stage.


#### 4.Geolocation Table Optimization Strategy

#### The Problem (Why it wasn't loaded with the other datasets)
The `olist_geolocation_dataset.csv` is an extremely heavy file containing **over 1 Million rows** of spatial coordinates ($Latitude$ and $Longitude$). 
* Loading this dataset initially with the other tables would unnecessarily drain system memory (RAM).
* **Data Redundancy:** Upon deep inspection, the dataset contains massive duplication where the exact same `zip_code_prefix` is repeated dozens of times with micro-differences in coordinates (just a few meters apart in the same street).
* Uploading 1M+ rows to SQL Server and feeding them into Power BI would severely degrade query performance and slow down dashboard rendering, without adding any real business value.

#### The Engineering Solution (Isolated Workflow)
To handle this efficiently, the geolocation dataset was isolated and treated separately using a **Data Aggregation Pipeline**:

1. **Spatial Grouping:** We performed a `.groupby()` on `geolocation_zip_code_prefix`.
2. **Coordinate Averaging:** We calculated the **Mean (Average)** of $Latitude$ and $Longitude$ for each unique ZIP code. This consolidates all scattered points into a single, highly accurate central point representing the entire neighborhood/area.
3. **Massive Size Reduction:** This optimization compressed the dataset from **1,000,000+ rows down to just ~19,000 unique rows** (one row per ZIP code).

#### Business & Dashboard Impact
* **100% Data Integrity:** No ZIP codes were deleted, and historic sales/revenue metrics remain completely untouched and accurate.
* **Blazing Fast Performance:** The compressed table `stg_geolocation` allows SQL Server to perform joins instantly and enables Power BI to render the **Geographical Heat Map** smoothly without lagging.

#### 5. Advanced Exploratory Analytics & Algorithmic Seller Segmentation (RFM)

Before migrating data into the warehouse, Python was leveraged as a diagnostic and predictive engine to answer critical strategic business questions:

##### A. Logistics & Customer Satisfaction Correlation Study
To understand the baseline operational performance and uncover what drives customer satisfaction, a multi-table inner join integrated orders, itemized financials, and review scores.
* **The Methodology:** Computed the exact historical delivery duration (`delivery_days`) per order. A **Spearman Rank Correlation Matrix** was computed across four key metrics (`delivery_days`, `freight_value`, `price`, `review_score`). Spearman was explicitly selected over Pearson to handle non-linear data distributions and minimize market outlier distortion.
* **Key Strategic Questions Answered:**
    * *Does shipping latency directly impact customer retention?* (Quantified the negative correlation between delivery days and review scores).
    * *Do expensive shipping fees (`freight_value`) systematically penalize seller ratings?*

##### B. Machine Learning-Driven Merchant Segmentation (K-Means)
To move beyond traditional, static business grouping that relies on biased human thresholds, an unsupervised machine learning pipeline was built to dynamically cluster the entire merchant base.

* **Feature Engineering (RFM):** Computed three core behavioral vectors per unique `seller_id`:
    * **Recency (R):** Days elapsed since the merchant’s latest transaction baseline.
    * **Frequency (F):** Total unique orders processed by the merchant.
    * **Monetary (M):** Total gross revenue generated by the merchant's transactions.
* **Relational Feature Scaling:** Applied standard normalization via `StandardScaler` to uniform the features (Mean = 0, Variance = 1). This mathematical scaling is crucial; without it, high-magnitude financial variables (`Monetary`) would completely overpower lower-magnitude transactional counts (`Frequency`), heavily distorting distance metrics within the cluster engine.
* **Algorithmic Clustering:** Deployed the **K-Means++** algorithm to isolate 4 distinct, highly cohesive merchant behavioral segments. 

##### The Business Translation Layer
The numerical cluster outputs (0, 1, 2, 3) were dynamically mapped into actionable, executive-friendly merchant profiles:
1. **Champions:** High-frequency, high-revenue merchants with recent transaction footprints. These drive platform growth and require VIP support and retention plans.
2. **Top Sellers:** Consistent, stable revenue generators maintaining steady operational frequency.
3. **Active (Low-Value):** High-volume or newly onboarded sellers pulling in low financial basket values per order; target candidates for cross-selling and volume-incentive programs.
4. **Lost/Hibernating:** High-recency dormant accounts that haven't processed an order in months. These indicate high merchant churn and require automated pipeline re-engagement campaigns.

> **End-to-End Pipeline Impact:** These ML-generated segments were seamlessly ingested through the SQL Warehouse layer and visualized on the Power BI dashboard (via the *Seller Segment Distribution* Donut Chart). This allows executive stakeholders to instantly filter the entire 100k-order marketplace ecosystem by merchant health with a single click.


### Phase 2: Spatial Optimization & Ingestion (Python to SQL)
* Developed an automated bulk upload pipeline using `SQLAlchemy` chunking to stream all 9 cleaned DataFrames into SQL Server simultaneously.

* **Database Architecture & Views:** Review the complete production DDL script, staging transformations, and optimized relational views: [`Olist_Star_Schema_and_Views.sql`](./sql/Olist_Star_Schema_and_Views.sql).

* **Artifact Reference:** View the full-resolution relational layout file directly in the repository: [`star_schema_diagram.png`](./sql/Star_schema_ER_digram.png).

### Database Architecture & Transformations (SQL Server)
Executed advanced T-SQL scripting directly in SSMS to transform raw staging data into an optimized **Star Schema Warehouse**:
1. **Relational Integrity:** Defined strict **Primary Keys (PK)** and **Foreign Keys (FK)** across all entities.
2. **Orphan Records Resolution:** Detected and handled 162 orphan ZIP codes found in Customer/Seller tables that didn't exist in the geolocation dataset using automated data imputation (Placeholder injection).
3. **Enterprise Date Dimension (`dim_date`):** Built a recursive CTE script generating a full 1,096-day calendar dimension (2016-2018) complete with Years, Quarters, Months, Weekdays, and Weekend flags for robust Time-Intelligence reporting.
4. **Abstraction via Database Views:** Created custom, optimized SQL Views (`v_dim_*`, `v_fact_*`) to handle text casting, type conversions, and embedded **English Translations** for all Portuguese product categories via `LEFT JOIN` and `ISNULL` functions.

### SQL Engineering Challenges & Solutions

During the database modeling phase in SSMS, several architectural bottlenecks and data alignment issues were encountered and resolved:

#### 1. The Geolocation Primary Key Conflict (`Nullable Column` Error)
* **The Problem:** SQL Server strictly prohibits enforcing a `PRIMARY KEY` constraint on columns that are marked as `Nullable` in their metadata, even if the underlying data has no missing values. The compressed `stg_geolocation` table initially rejected the PK constraint on `zip_code_prefix`.
* **The Solution:** Isolated the metadata update using explicit `ALTER COLUMN ... INT NOT NULL` commands, paired with the **`GO`** batch separator. This forced SSMS to commit the structural schema change into database memory before immediately applying the `ADD CONSTRAINT` command in the subsequent batch.

#### 2. Referential Integrity & The Orphan Records Crisis (SQL Error 547)
* **The Problem:** When attempting to establish `FOREIGN KEY` relationships from `stg_customers` and `stg_sellers` to `stg_geolocation`, the server threw a hard conflict error (Msg 547). This indicated that there were **162 unique legacy ZIP codes** present in the operational transaction tables that did not exist anywhere in the geographical dataset.
* **The Solution (Data Imputation via SQL):** Instead of compromising the database security by skipping constraints, an automated SQL insertion script was deployed. It scanned both tables using `NOT IN` subqueries, dynamically isolated the missing ZIP codes, and injected them into `stg_geolocation` using a standardized **Placeholder Architecture** (`(0, 0)` for coordinates, and `'Unknown'` for city/state strings). This safely retained 100% of customer orders while securing the Star Schema's referential integrity.

#### 3. Cross-Table Data Type Mismatches
* **The Problem:** Columns acting as relational keys across tables were uploaded from Python with incompatible data types (e.g., Zip Codes as `BIGINT` in transactional tables vs. `INT` in the aggregated geolocation table), preventing any indexing or constraint building.
* **The Solution:** Executed strict data type alignment using targeted T-SQL `ALTER` scripts to uniform all keys into optimal structural types before attempting any table joins.
---

#### Data Model / Star Schema Architecture
The database infrastructure is fully decoupled into **Fact** and **Dimension** layers through the generated SQL views:

* **Fact Tables:**
  * `v_fact_orders` (Core transactions & timestamps)
  * `v_fact_order_items` (Financials, prices, and freight)
  * `v_fact_payments` (Payment types and installment structures)
  * `v_fact_reviews` (Customer satisfaction metrics)
* **Dimension Tables:**
  * `v_dim_customers` (Unique client mapping)
  * `v_dim_products` (Cleaned, translated, and typed product specs)
  * `v_dim_sellers` (Merchant distribution)
  * `v_dim_geolocation` (Compressed central spatial coordinates)
  * `dim_date` (Time-intelligence calendar)

  > **Architectural Note on Geolocation:** As shown in the SSMS ER Diagram, `stg_geolocation` exists as a single consolidated storage table to maintain database normalization and reduce storage footprint. However, to prevent relationship ambiguity inside the Power BI BI Engine, this table is logically decoupled at the Semantic Layer into two role-playing tables: `v_customers_dim_geolocation` and `v_sellers_dim_geolocation`.
---

### Phase 3: BI Reporting & Value Creation (Power BI Preview)

[Production Power BI Report](./power_bi/Olist.pbix)

### Dashboard Previews

#### 1. Sales & Marketplace Performance
![Sales Dashboard](./power_bi/Sales & Marketplace Performance.png)

#### 2. Logistics & Delivery Performance
![Logistics Dashboard](./power_bi/Logistics & Delivery Performance.png)

#### 3. Seller Order-Level Performance (Drillthrough Audit)
![Drillthrough Dashboard](./power_bi/Seller Order-Level Performance.png)

* **Data Modeling & BI Architecture:** Imported optimized SQL Server database views to construct a robust Star Schema. Modeled relationships with strategic filter propagation (including advanced `CROSSFILTER` implementations to handle order-level vs. seller-level granularities seamlessly).

* **Power Query ETL Governance & Applied Steps Documentation:** Conducted the final data transformation layer using Power Query, ensuring optimal memory management. Adhering to enterprise-grade development standards, **every single "Applied Step" across all queries was explicitly renamed and fully documented** to explicitly reference the exact transformation and data cleaning logic applied, eliminating black-box processing for long-term maintainability.

* **Semantic Layer Spatial Optimization (Role-Playing Dimensions):** While the aggregated geolocation dataset was stored as a single consolidated table in SQL Server to optimize database storage, it was decoupled into two distinct semantic tables inside Power Query to eliminate relationship ambiguity inside the BI Engine:
    * `v_customers_dim_geolocation`: Mapped directly to customer demographic dimensions for buyer spatial analytics.
    * `v_sellers_dim_geolocation`: Mapped exclusively to seller profiles to drive logistics and merchant distribution heatmaps independently.
    * **Algorithmic Merchant Dimension Swap:** Replaced the static operational `stg_sellers` dimension table with the enriched machine-learning-derived `dim_sellers_rfm` dimension. This architectural swap embeds dynamic behavioral profiles (*Champions*, *Top Sellers*, *Active*, *Lost*) into the core data model, shifting the dashboard from generic operational tracking to advanced merchant-health analytical segmentations.

* **Advanced DAX & Metrics Engineering :** Built out complex time-intelligence and operational DAX measures (e.g., dynamic *Average Delivery Days*, YoY Growth, and automated KPI aggregations).
For a complete formulas repository, architectural business logic, and exact DAX implementations of all explicit measures, check out the standalone [`Dax_measurements_dictionary.md`](./power_bi/Dax_measurements_dictionary.md) handbook.

* **Enterprise UI/UX Dashboard Design:** Developed a multi-page, high-fidelity dark-themed report featuring dynamic visual storytelling across three core layers:
    
    * *Sales & Marketplace Performance:* High-level overview of revenue trends, product category share, and global geographical distribution, augmented with a custom Donut Chart for **Seller Segment Distribution**.
    
    | KPI Card Metric | Technical DAX Context / Business Value |
    | :--- | :--- |
    | (Total Sales) | Aggregate gross revenue generated across the marketplace (sum of product prices), serving as the core top-line growth metric. |
    | (Total Orders) | Total volume of unique, finalized customer transaction loops across the entire Olist platform history. |
    | (Avg Order Value) | Calculated dynamically as `Total Sales / Total Orders`. Represents the average basket size spend per checkout transaction. |
    | (Active Sellers) | Count of unique, distinct merchant IDs that successfully transacted and fulfilled at least one order during the operational lifecycle. |

    *(Augmented with a custom Donut Chart for **Seller Segment Distribution** utilizing native Header Icons for help context tooltips on-hover).*    

    * *Logistics & Delivery Performance:* Deep dive into delivery timelines, carrier efficiencies, and shipping costs.
    
    | KPI Card Metric | Technical DAX Context / Business Value |
    | :--- | :--- |
    | (Avg Review Score) | The platform-wide customer satisfaction benchmark, dynamically tracked to measure the direct correlation between shipping speeds and post-delivery review scores. |
    | (Total Items Sold) | The cumulative volume of physical line-items processed and shipped through Olist logistics nodes (accounting for multi-item orders). |
    | (Avg Delivery Days) | The mean duration calculated from order approval timestamp to the final customer delivery confirmation, identifying core fulfillment velocity. |
    | (Delayed Orders %) | A critical SLA compliance metric calculated as `TOTAL(Delayed Orders) / Total Orders`. Tracks the percentage of shipments missing their strict promised delivery dates. |

    *(Augmented with advanced spatial charts and a dynamic bubble analysis crossing Customer Sentiment against Freight-to-Sales ratios across product categories).*

    * *Seller Order-Level Performance (Drillthrough):* An operational audit screen enabling stakeholders to deep-dive into an individual seller's full order checklist, review scores, and financial metrics with a single click.
    
    | KPI Card Metric | Technical DAX Context / Business Value |
    | :--- | :--- |
    |  (Total Sales) | Isolated gross revenue generated exclusively by the selected merchant, allowing instant tracking of individual vendor contribution. |
    | (Total Items Sold) | The specific volume of physical products successfully distributed and sold by this individual seller. |
    | (Avg Review Score) | The localized reputation score for the selected vendor, critical for monitoring individual merchant quality and marketplace SLA compliance. |
    | (Total Freight Cost) | The cumulative logistics and shipping fees generated by this seller's orders, helping evaluate their operational freight efficiency. |

    *(Augmented with a low-level transactional ledger detailing exact `order_id` strings, localized `product_category` classifications, and individual `Order Review Scores` for laser-focused root-cause analysis).*

* **Global Filter Synchronization & Contextual Help System:** 
    * **Cross-Report Global Slicers (comparing behavior across multiple product categories simultaneously):** All dashboard pages feature fully synchronized cross-filtering panels, enabling end-users to seamlessly slice the entire marketplace ecosystem dynamically by **Year, Product Category, Merchant RFM Segment, and Brazilian State**.
    * **Universal Contextual Documentation:** To ensure self-service BI adoption, **every single visual chart incorporates a standardized Question Mark (`?`) helper icon**. On-hover tooltip actions are fully configured across these header icons, providing non-technical stakeholders with immediate contextual business definitions and formula logic behind each metric.