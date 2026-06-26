### 1. Customers Table (`olist_customers_dataset.csv`)
This table contains customer demographic data and serves as the primary anchor for identifying unique buyers and mapping their geographical distribution.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `customer_id` | Text (Alphanumeric) | A unique key generated **per order**. Each time a user makes a purchase, a new `customer_id` is created (used to join with the Orders table). |
| `customer_unique_id` | Text (Alphanumeric) | The **permanent identifier** for the actual customer. This key allows tracking repeat purchases and analyzing long-term Customer Lifetime Value (CLV). |
| `customer_zip_code_prefix` | Text / Number | The first 5 digits of the customer's postal code, mapping their physical location. |
| `customer_city` | Text | The city name where the customer is located. |
| `customer_state` | Text (2-letter code) | The Brazilian state abbreviation where the customer resides (e.g., SP, RJ). |

### 2. Orders Table (`olist_orders_dataset.csv`)
This is the core fact table capturing the full operational lifecycle, milestone timestamps, and fulfillment statuses for every unique transaction on the platform.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `order_id` | Text (Alphanumeric) | The unique primary key identifying each specific order placed on the platform. |
| `customer_id` | Text (Alphanumeric) | Foreign key linking to the Customers table (maps to the order-specific `customer_id`). |
| `order_status` | Text | The current phase of the order lifecycle (e.g., `delivered`, `shipped`, `invoiced`, `processing`, `canceled`). |
| `order_purchase_timestamp` | DateTime | The exact timestamp when the customer finalized checkout on the platform. |
| `order_approved_at` | DateTime | The timestamp confirming payment validation and authorization. |
| `order_delivered_carrier_date` | DateTime | The timestamp when the merchant handed off the packed parcel to the logistics carrier. |
| `order_delivered_customer_date` | DateTime | The actual final date and time when the parcel reached the customer's doorstep. |
| `order_estimated_delivery_date` | DateTime | The target SLA delivery date promised to the customer during checkout. |

### 3. Order Items Table (`olist_order_items_dataset.csv`)
This table contains the line-item granularity of each purchase, mapping every product inside an order to its specific merchant, pricing, and logistical cost.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `order_id` | Text (Alphanumeric) | Foreign key mapping back to the primary transactional order envelope. |
| `order_item_id` | Integer | A sequential number identifying the items or item count within the same order block (e.g., if an order has 3 items, values will be 1, 2, 3). |
| `product_id` | Text (Alphanumeric) | Foreign key linking to the commercial products dimension catalog. |
| `seller_id` | Text (Alphanumeric) | Foreign key linking to the specific merchant responsible for fulfilling this product. |
| `shipping_limit_date` | DateTime | The strict SLA deadline set by Olist for the merchant to hand over the product to the logistics partner. |
| `price` | Decimal | The actual selling price per individual item (excluding shipping fees). |
| `freight_value` | Decimal | The specific logistics and delivery fee charged per item based on physical distance and parcel dimensions. |

### 4. Order Payments Table (`olist_order_payments_dataset.csv`)
This table details the financial transaction layers for each order, capturing payment methods, installment structures, and total monetary values.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `order_id` | Text (Alphanumeric) | Foreign key mapping back to the primary transactional order envelope. |
| `payment_sequential` | Integer | A sequential number identifying multiple payment methods used for a single order (e.g., if a customer pays with two different credit cards, values will be 1, 2). |
| `payment_type` | Text | The financial instrument used for the transaction (e.g., `credit_card`, `boleto` [Brazilian voucher], `voucher`, `debit_card`, `not_defined`). |
| `payment_installments` | Integer | The total number of installment slices chosen by the customer for credit card financing. |
| `payment_value` | Decimal | The total monetary amount charged and settled for this specific payment record. |

### 5. Products Table (`olist_products_dataset.csv`)
This dimension table acts as the master product catalog, housing metadata, classifications, and physical dimensions for all items sold on the marketplace.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `product_id` | Text (Alphanumeric) | The unique primary key identifying each specific product cataloged on the platform. |
| `product_category_name` | Text | The root category name of the product in Portuguese (e.g., `beleza_saude`, `automotivo`). |
| `product_name_lenght` | Integer | The total character count of the product's commercial title. |
| `product_description_lenght`| Integer | The total character count of the product's detailed description text. |
| `product_photos_qty` | Integer | The total number of marketing images published on the product's marketplace page. |
| `product_weight_g` | Integer | The physical weight of the product measured in grams (critical for freight calculations). |
| `product_length_cm` | Integer | The physical length of the product package measured in centimeters. |
| `product_height_cm` | Integer | The physical height of the product package measured in centimeters. |
| `product_width_cm` | Integer | The physical width of the product package measured in centimeters. |

### 6. Product Category Name Translation Table (`product_category_name_translation.csv`)
This lookup reference table provides English translations for the Portuguese product category names, enabling standardized global reporting.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `product_category_name` | Text | The root category name of the product written in Portuguese (maps directly to the Products table). |
| `product_category_name_english` | Text | The localized English translation of the product category name (e.g., `health_beauty`, `auto`). |

### 7. Order Reviews Table (`olist_order_reviews_dataset.csv`)
This table contains performance feedback provided by customers post-delivery, capturing satisfaction scores and raw textual review comments.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `review_id` | Text (Alphanumeric) | The unique primary key identifying each specific review submission. |
| `order_id` | Text (Alphanumeric) | Foreign key mapping back to the primary transactional order envelope. |
| `review_score` | Integer (1 to 5) | Satisfaction rating given by the customer, ranging from 1 (highly dissatisfied) to 5 (highly satisfied). |
| `review_comment_title` | Text | The short heading or title written by the customer for their review (often empty). |
| `review_comment_message` | Text | The detailed textual comment or feedback feedback left by the customer regarding their experience. |
| `review_creation_date` | DateTime | The timestamp when the automated survey notification was dispatched to the customer. |
| `review_answer_timestamp` | DateTime | The exact timestamp when the customer submitted their final review response. |

### 8. Sellers Table (`olist_sellers_dataset.csv`)
This dimension table houses master records for all registered marketplace merchants, mapping their physical fulfillment hubs and regional locations.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `seller_id` | Text (Alphanumeric) | The unique primary key identifying each specific seller/merchant on the platform. |
| `seller_zip_code_prefix` | Text / Number | The first 5 digits of the seller's postal code, identifying their fulfillment center location. |
| `seller_city` | Text | The city name where the merchant's storefront/warehouse is registered. |
| `seller_state` | Text (2-letter code) | The Brazilian state abbreviation where the merchant operates. |

### 9. Geolocation Table (`olist_geolocation_dataset.csv`)
This spatial reference table maps Brazilian postal code prefixes to precise geographic coordinates, enabling spatial analytics and distance-based logistics modeling.

| Column Name | Data Type | Description / Business Logic |
| :--- | :--- | :--- |
| `geolocation_zip_code_prefix` | Text / Number | The first 5 digits of the postal code (Primary spatial key used to link with Customers and Sellers tables). |
| `geolocation_lat` | Decimal | The precise latitude coordinate for the center of the postal code prefix zone. |
| `geolocation_lng` | Decimal | The precise longitude coordinate for the center of the postal code prefix zone. |
| `geolocation_city` | Text | The localized city name mapped to the specific spatial coordinates. |
| `geolocation_state` | Text (2-letter code) | The Brazilian state abbreviation associated with the geographical record. |

---

[Back to Main README](./README.md)
