## Phase 2: Data Warehousing, Relational Architecture & DDL Transformations

---
**Visual Schema Verification:** To view the complete physical ER layout and visual entity relationships generated directly from SQL Server, check out the architecture map here: [`star_schema_diagram.png`](https://github.com/Seif-Fathi/Olist-E2E-Data-Engineering-BI/blob/main/sql/Star_schema_ER_digram.png).

Adhering to enterprise relational database design standards, the raw staging layer (`stg_`) was systematically re-architected inside SQL Server (SSMS) into a high-performance **Star Schema**. This phase guarantees data integration, mathematical lineage stability, and sub-second query execution loops during Power BI ingestion.

The complete production DDL script, constraint mappings, and optimized views are cataloged in [`Olist_Star_Schema_and_Views.sql`](https://github.com/Seif-Fathi/Olist-E2E-Data-Engineering-BI/blob/main/sql/Olist_Star_Schema_and_Views.sql).

---

### 1. Primary & Composite Key Matrix (Referential Integrity)
To transition from flat CSV logs into a structured data warehouse, explicit physical constraints were enforced across the database engine:
* **Dimension Table Enforcements:** Unique alphanumeric constraints were injected to establish Primary Keys across `stg_customers`, `stg_products`, and `stg_sellers` using `ALTER TABLE` DDL operations.
* **Granular Geolocation Uniqueness:** Leveraging Python-pre-aggregated prefixes, `zip_code_prefix` was safely promoted to a non-nullable Primary Key inside `stg_geolocation`.
* **Composite Key Engineering:** For the granular line-item tracking table (`stg_order_items`), strict logical granularity was established by constructing a Composite Primary Key combining (`order_id` + `order_item_id`).

---

### 2. Architectural Challenge: Resolving Orphan Records & Referential Violations
During the automated Foreign Key propagation step, the relational engine rejected constraint enforcement between the Geolocation table and the Customer/Seller dimensions due to severe data quality leakage in the production source.

* **The Problem (Data Anomaly):** The operational tables contained **Orphan ZIP Codes**—valid transactional customer and merchant postal prefixes that did not exist in the primary spatial reference lookup table (`stg_geolocation`), triggering immediate `FOREIGN KEY` constraint violations.
* **The Business Logic Resolution (Data Imputation):** Rather than performing destructive queries (deleting unmatched production transactions, which would artificially shrink total revenue figures), a programmatic **SQL Data Imputation** loop was designed:
  ```sql
  INSERT INTO stg_geolocation (zip_code_prefix, lat, lng, city, state)
  SELECT DISTINCT customer_zip_code_prefix, 0, 0, 'Unknown', 'Unknown'
  FROM stg_customers
  WHERE customer_zip_code_prefix NOT IN (SELECT zip_code_prefix FROM stg_geolocation);

The Result: Missing indices were dynamically populated with spatial coordinate placeholders (0, 0) and semantic identifiers ('Unknown'). This successfully restored downstream lineage integrity and enabled flawless Foreign Key constraint deployment without losing a single cent of historical transaction data.

### 3. Dynamic Date Dimension Engineering (dim_date)
To empower robust Time-Intelligence reporting (YoY Growth, Quarterly trends, and Weekend vs. Weekday order behaviors), a dedicated Temporal Dimension was developed from scratch rather than relying on automated Power BI calendar auto-generation.

Algorithmic Population: Utilized a high-performance Recursive CTE (Common Table Expression) to dynamically generate daily calendar slices spanning from 2016-01-01 to 2018-12-31, matching the exact operational spectrum of Olist.

Constraint Configuration: Enforced OPTION (MAXRECURSION 0) to bypass SQL Server's default iteration safety threshold, enabling continuous 3-year record generation. The stg_orders core timestamp was then downcast to a standard DATE type to seal a clean, indexed relationship with dim_date(date_key).

### 4. Semantic Layer Decoupling via Database Views (v_)
To safeguard the underlying physical tables and decouple raw storage formatting from BI ingestion, an abstraction semantic layer consisting of 8 optimized Database Views was deployed.

Data Ingestion & Schema Transformation Rules:
Dynamic Language Localization (v_dim_products): Embedded a LEFT JOIN against the stg_translations lookup table utilizing ISNULL(). This seamlessly swaps native Portuguese category text with clean English strings (product_category_name_english) directly at the server level.

Explicit Data Type Sanitization: Enforced strict data cast layers (e.g., casting operational strings into strict DATETIME, FLOAT, and INT metrics) within the views. This strips trailing spaces, flags corruption early, and optimizes memory allocation for the Power BI VertiPaq columnar compression engine.

Advanced Merchant-Health Segmentation (v_dim_sellers_enriched): Programmatically merged the basic merchant profiles with the machine-learning-derived algorithmic table dim_sellers_rfm. This natively surfaces dynamic behavioral cohorts (Recency, Frequency, Monetary, and Seller_Segment) inside a single, enriched dimension view.

### 5. Relational Architecture & Schema Mapping
The core database design is mapped below, explicitly showcasing how the Fact and Dimension tables interact through strict Primary Key (PK) and Foreign Key (FK) constraints:

**v_fact_order_items (Order Items Line-Granularity Fact):**

* Connected to v_fact_orders via order_id (FK $\rightarrow$ PK). Relationship: Many-to-One (Handles multi-item orders).

* Connected to v_dim_products via product_id (FK $\rightarrow$ PK). Relationship: Many-to-One.

* Connected to v_dim_sellers_enriched via seller_id (FK $\rightarrow$ PK). Relationship: Many-to-One.

**v_fact_payments (Payments Fact):**

* Connected to v_fact_orders via order_id (FK $\rightarrow$ PK). Relationship: Many-to-One (Handles split payment methods per order).

**v_fact_reviews (Customer Reviews Fact):**

* Connected to v_fact_orders via order_id (FK $\rightarrow$ PK). Relationship: Many-to-One.

2. Spatial / Role-Playing Dimension Mapping

**v_dim_geolocation (Master Spatial Lookup):**

Acts as a shared reference boundary for both buyers and merchants.

* Connected to v_dim_customers via zip_code (PK $\rightarrow$ FK). Relationship: One-to-Many.

* Connected to v_dim_sellers_enriched via zip_code (PK $\rightarrow$ FK). Relationship: One-to-Many.

3. Enriched Behavioral Dimensions

**v_dim_sellers_enriched (Advanced Merchant Profile):**

Formed via a physical server-side INNER JOIN between stg_sellers and dim_sellers_rfm on seller_id.

This embeds algorithmic machine-learning clusters (Champions, At Risk, Lost) directly into the dimensional schema without breaking legacy relational constraints.




