## **Northline Outfitters — Project 2**
Group B4 

## Table of Contents
1. [Team Members / Roles](#team-members--roles)
2. [Case Summary](#case-summary)
3. [Conceptual Model / Database Description](#conceptual-model--database-description)
4. [Data Model](#data-model)
5. [Database Description](#database-database-description)
6. [Data Cleaning Process: Product_Supplier_Master](#data-cleaning-process-product_supplier_master)
7. [Data Cleaning Process: Sales_Dump](#issues-identified)
8. [SQL Code: Sales_Dump Cleaning](#sql-code-used-to-clean-sales_dump)
9. [SQL Code: Product_Supplier_Master Cleaning](#sql-codes-to-clean-the-product_supplier_master-table)
10. [Justification for Cleaning Steps](#justification-for-cleaning-steps)
11. [Queries](#queries)

## **Team Members / Roles**

Hiya Shah (Data Wrangler), Morgan Matherne (Database Designer), Mark Monzer (Conceptual Modeler), Roshan Gadiraju (SQL Writer), Zeynep Koseoglu (Group Leader)


## **Case Summary**

Northline Outfitters is a small online retail company that sells student-focused lifestyle and technology accessories, including items such as hoodies, water bottles, desk lamps, phone cases, keyboards, mouse pads, and backpacks. T

The company provided two primary datasets: a Sales_Dump spreadsheet containing transaction-level sales data and a Product_Supplier_Master spreadsheet containing product and vendor information. These datasets reflect real-world data challenges commonly found in small and medium-sized businesses. The data was intentionally messy, unnormalized, and inconsistent, including issues such as mixed date formats, inconsistent category labels, embedded currency values, duplicate product entries, unstructured customer information, and measurements recorded in both metric and imperial units.

The objective of this project was to transform these raw spreadsheets into a clean, structured relational database that supports business analysis. This involved identifying key data quality issues, cleaning and standardizing the data using SQL, and designing a normalized conceptual model that separates the data into meaningful entities such as products, vendors, customers, orders, and order lines. The final database was designed to reduce redundancy, improve data consistency, and enable efficient querying.

The cleaned and structured database supports important business questions for Northline Outfitters, such as identifying top-selling products by country, evaluating employee performance in handling orders, and analyzing vendor-product relationships across categories. By converting the original spreadsheets into a relational model, the company gains a more reliable and scalable data foundation for decision-making, similar to what would be expected in a real-world business environment transitioning from spreadsheet-based operations to a database system.

## **Conceptual Model / Database Description**
## Data Model
![Project Image](mistproject2.png)

## Database Database Description

Our database was designed to transform the two messy source spreadsheets, Sales_Dump and Product_Supplier_Master, into a compact relational model that supports data cleaning, structured storage, and the required SQL analysis. 

The model centers on the retail sales process for Northline Outfitters, a small online retailer that purchases products from outside vendors and sells them directly to customers in the United States and Canada. The database separates operational data into core business entities so that customer, employee, product, vendor, order, and payment information are not stored repeatedly in a single spreadsheet row. This reduces redundancy and makes the data easier to query and maintain. T

| Entity | Type | Key Attributes | Relationships |
|---|---|---|---|
| **Orders** | Transactional | order_id (PK), sale_date, shipping_location, country | Belongs to 1 Customer; handled by 1 Employee; has many OrderLines; uses 1 PaymentMethod |
| **OrderLines** | Transactional | line_id (PK), order_id (FK), sku (FK), quantity, unit_price, discount_amt, tax_amt, line_total, return_status, size, weight | Belongs to 1 Order; references 1 Product |
| **Product** | Reference | sku (PK), alt_sku, parent_sku, description, list_price, discontinued, weight, length, reorder_level, pack_size, notes, category_id (FK) | Appears in many OrderLines; belongs to 1 Category; linked to Vendors via product_vendor |
| **Categories** | Lookup | category_id (PK), category_name | Classifies many Products |
| **Vendor** | Reference | vendor_id (PK), vendor_name, phone, vendor_rep | Linked to Products via product_vendor |
| **product_vendor** | Bridge | sku (FK), vendor_id (FK), cost | Connects Products to Vendors; stores vendor-specific cost |
| **Customer** | Reference | customer_id (PK), name, email, customer_type, notes, loyalty_info | Places many Orders |
| **Employees** | Reference | employee_id (PK), manager_id (FK self-ref) | Handles many Orders; manager_id supports hierarchical reporting |
| **PaymentMethods** | Lookup | payment_method_id (PK), payment_type | Used by many Orders |


The model separates transactional data into Orders and OrderLines. Orders store order-level details (date, customer, employee, payment, location), while OrderLines store product-level details (SKU, quantity, price, discount, tax, returns). This prevents repeating groups and supports accurate sales and revenue analysis.

The Product table stores all product attributes (SKU, description, price, size/weight, category, etc.) and includes parent_sku to handle product variants. Product data is separated from transactions so it is maintained once and linked through SKU.

Supporting tables include Category, Vendor, and a product_vendor bridge table. Categories standardize product groupings, while Vendor stores supplier information. The bridge table allows products to be linked to vendors and supports more flexible supplier relationships.

Customer and employee data were separated into Customer and Employees tables to reduce duplication from the original dataset and support performance analysis. PaymentMethods was also separated to standardize inconsistent payment values.

## **Data Cleaning Process: Product_Supplier_Master**

The Product_Supplier_Master dataset contained multiple data quality issues, including inconsistent identifiers, mixed text and numeric formats, embedded currency labels, inconsistent units of measure, redundant helper columns, and unstructured notes. These issues were cleaned using SQL so the data could be standardized and loaded into the final relational model. The source spreadsheet included product, vendor, pricing, packaging, measurement, discontinuation, and parent SKU fields, so the cleaning process focused on making each of those attributes consistent and usable in the database.

**Our Process**
We imported the two sheets into MySQL and used the following queries to clean the data and address the issues identified above. We then pulled the cleaned data from the tables and split it into 8 sheets that correspond to our data model entities and the columns generated by forward-engineering our data model. Any columns we had difficulty cleaning in my SQL, we manually fixed in Excel. An example of this was the pack size; we changed all the different formatting presentations to just the number.

## Issues Identified
## `Sales_Dump`

### 1. `line_id`
**Purpose:** Unique identifier for each line item (e.g., `LN-001`)

**Issues:**
- No issues detected. All values follow the `LN-###` format and appear unique.

---

### 2. `order_id`
**Purpose:** Links line items to a parent order (e.g., `UORD-1041`, `CORD-1003`)

**Issues:**
- **Same order ID appears on multiple rows** — expected for multi-line orders, but some orders appear across inconsistent dates (see `sale_date` issues below).
- **Two prefixes used (`UORD-` vs `CORD-`)** — presumably US vs. Canada orders, but this distinction is never documented and inconsistently applied (some CA-country rows use `UORD-`, and vice versa).

```
LN-054: CORD-1032 | ship_country = blank (missing)
LN-057: CORD-1041 | ship_country = blank (missing)
LN-066: UORD-1052 | ship_country = blank (missing)
```

---

### 3. `sale_date`
**Purpose:** Date the sale occurred

**Issues:**
- **At least 7 different date formats used** — no consistent standard:

| Format Example       | Row(s)       |
|----------------------|--------------|
| `10-11-2025`         | LN-001       |
| `Oct 17 25`          | LN-002       |
| `October 5 25`       | LN-003       |
| `June 11 2025`       | LN-004       |
| `10/14/2025`         | LN-014       |
| `31/10/2025`         | LN-017       |
| `10 Sep 2025`        | LN-010       |

- **Same order with different date formats** — e.g., `UORD-1095` (LN-166, LN-167, LN-168) uses three different formats for the same date.
- **Future date found** — LN-191 has `May 1 2026`, which is outside the dataset's 2025 scope.
- **Ambiguous MM/DD vs DD/MM** — formats like `08/10/2025` (LN-105) could be August 10 or October 8 depending on locale convention.

---

### 4. `employee_ref`
**Purpose:** ID of the employee who processed the sale (e.g., `EMU-202`)

**Issues:**
- No format issues detected. All values follow a consistent `EM?-###` pattern.
- **Undocumented prefix convention:** `EMU-` vs `EMC-` appears to distinguish US vs. Canada employees, but this is not documented and the relationship to `ship_country` is not enforced.

---

### 5. `manager_ref`
**Purpose:** ID of the supervising manager (e.g., `EMU-M03`)

**Issues:**
- No format issues detected.
- Same undocumented `EMU-M` vs `EMC-M` prefix concern as `employee_ref`.

---

### 6. `customer_info`
**Purpose:** Customer name and optional attributes like loyalty status or customer type

**Issues:**
- **Multiple delimiter styles used** — semicolons (`;`), pipes (`|`), and slashes (`/`) all appear within the same column:
  - `Mason Rivera; Loyalty? Y` (LN-001)
  - `Grace Hall | Student | US` (LN-003)
  - `Zoe Garcia / guest order` (LN-009)
- **Multiple attributes packed into one field** — loyalty status, student flag, country, and guest/registered status are all embedded in free text rather than stored in separate columns.
- **Same customer appears in multiple formats** across rows with no canonical form.
- **No null values**, but data is not parseable without custom string logic.

---

### 7. `customer_email`
**Purpose:** Customer's email address

**Issues:**
- **Many blank/missing values** — guest orders and some loyalty customers have no email (LN-008, LN-011, LN-016, LN-023, LN-031, and many more).
- **Invalid email formats:**
  - LN-017: `emma_wilson@school.ed` — TLD is truncated
  - LN-022: `ella.wright@example.caa` — extra character in TLD
- **Same customer with different emails** — e.g., Grace Hall appears as `gracehall@gmail.com`, `grace_hall@school.edu`, and `grace.hall@example.com` across rows with no reconciliation.
- **Inconsistent use of personal vs. institutional email** for the same customer on different orders.

---

### 8. `payment_method`
**Purpose:** Method used to pay (e.g., VISA, Debit, Cash)

**Issues:**
- **Inconsistent capitalization** — the same method recorded differently across rows:
  - `VISA` / `visa` / `Visa`
  - `Debit` / `debit`
  - `MC` / `Mastercard` / `MasterCard`
- **Non-standard abbreviation** — `MC` for Mastercard is used inconsistently alongside the full name.
- **`AMEX`** appears only in LN-008 and LN-146 — may be valid but is not in any documented allowable-values list.
- **`Interac`** appears for some rows where `Debit` might be expected; relationship between the two is undefined.

---

### 9. `sku`
**Purpose:** Stock-keeping unit identifier for the product (e.g., `SKU-C-1014`)

**Issues:**
- **Inconsistent capitalization** — some SKUs are entirely lowercase:
  - `sku-c-1012` (LN-006), `sku-u-1019` (LN-013), `sku-u-1011` (LN-048), `sku-c-1020` (LN-062), `sku-u-1001` (LN-080), etc.
- The same product can appear under both cased variants, making joins or lookups unreliable without normalization.

---

### 10. `product_description`
**Purpose:** Human-readable product name

**Issues:**
- **Inconsistent capitalization** — the same product appears in title case and ALL CAPS in different rows:

| Product              | Title Case              | ALL CAPS                 |
|----------------------|-------------------------|--------------------------|
| Breeze Ring Light    | LN-015                  | LN-002, LN-106           |
| CloudSleeve Laptop Case | LN-010               | LN-033, LN-132           |
| Summit Laptop Stand  | LN-042                  | LN-041, LN-102           |
| InkTrail Journal     | LN-004                  | LN-017, LN-143           |
| Glide Mouse Pad      | LN-011                  | LN-064, LN-073           |
| Halo Desk Lamp       | LN-027                  | LN-066, LN-068           |
| SnapCharge Cable     | LN-097                  | LN-074, LN-091, LN-128   |
| Studio Webcam Cover  | LN-023                  | LN-013, LN-195           |

- Description should be derived from SKU lookup, not entered as free text.

---

### 11. `category`
**Purpose:** Product category classification

**Issues:**
- **Same product assigned different categories across rows** — the `/ Student` suffix is applied inconsistently for the same SKU:
  - `SKU-C-1018` (CloudSleeve): `Accessories / Student` (LN-008) vs. `Accessories` (LN-028)
  - `SKU-U-1013` (InkTrail Journal): `School` (LN-016) vs. `School / Student` (LN-022)
  - `SKU-U-1005` (Halo Desk Lamp): `Desk Setup` (LN-120) vs. `Desk Setup / Student` (LN-066)
- **No controlled vocabulary** — the student designation is not a separate column; it is appended as a suffix with no enforcement.
- **Inconsistent spacing** around the `/` separator.

---

### 12. `quantity`
**Purpose:** Number of units sold

**Issues:**
- **Mixed data types** — some values are plain integers, others include the word "units":
  - Integer: `1`, `2`, `3`
  - String: `2 units` (LN-014), `3 units` (LN-025), `2 units` (LN-043), `2 units` (LN-047), etc.
- This makes the column non-numeric and prevents direct aggregation or arithmetic without cleaning.

---

### 13. `unit_price`
**Purpose:** Per-unit price of the product

**Issues:**
- **Currency code embedded in value** — `USD 18.99`, `CAD 46.99` — the currency prefix makes every value a string rather than a number.
- **No dedicated currency column** — currency is only determinable by parsing the price string; there is no separate `currency` field.
- **Same SKU priced differently across rows** — price variation exists (possibly by region or time) but is never explained or documented.

---

### 14. `discount`
**Purpose:** Discount applied to the line item

**Issues:**
- **Multiple formats for the same concept — cannot be parsed as a number without normalization:**
  - Percentage with symbol: `10%`, `5%`
  - Raw number: `5`, `10`, `0`
  - Named discount codes: `promo5`, `student 10%`
  - Blank (missing) and `0` and `0%` all used to represent "no discount"
- A value of `5` is ambiguous — it could mean 5%, $5 off, or something else.

---

### 15. `tax`
**Purpose:** Tax rate or tax amount applied

**Issues:**
- **Three different representations of the same rate — impossible to parse consistently:**
  - Decimal rate: `0.07`, `0.13`, `0.0825`
  - Percentage string: `7%`, `13%`, `8.25%`
  - Labeled string: `HST 13%`
- **Many blank values** — LN-001, LN-003, LN-007, LN-023, and many others have no tax recorded.
- **Ambiguity between rate and amount** — `0.07` could mean a 7% tax rate or a $0.07 flat tax; context does not clarify which.

---

### 16. `line_total`
**Purpose:** Total dollar amount for the line item

**Issues:**
- **Inconsistent dollar sign formatting** — dollar sign present in some values, absent in others:
  - `$19.43` (LN-001) vs. `19.43` (LN-098)
  - `$32.38` (LN-009) vs. `32.38` (LN-016)
- **Many blank values** — LN-002, LN-003, LN-004, LN-005, LN-006, and dozens more are missing a `line_total`.
- **Calculation errors** — in some rows where all inputs are present, the `line_total` does not match the expected result. For example:
  - LN-101: qty=1, price=USD 50.99, discount=5, tax=8.25% → expected ~$52.17, but `line_total` = 161.97 (appears to be the 3-unit total from a different row)

---

### 17. `ship_country`
**Purpose:** Country the order ships to

**Issues:**
- **Inconsistent country code vs. full name:**
  - `US` (majority) vs. `USA` (LN-044, LN-045, LN-081, LN-082)
  - `CA` (majority) vs. `Canada` (LN-042, LN-067, LN-079)
- **Blank values** — LN-054, LN-057, LN-066 are missing ship country entirely.

---

### 18. `ship_to`
**Purpose:** Shipping destination

**Issues:**
- **Non-address placeholder values used instead of real addresses:**
  - `Same as billing` — billing address not stored in this dataset
  - `Dorm pickup` — no location data at all
- **Inconsistent specificity** — some rows have `City, Province/State` (e.g., `Toronto, ON`, `Seattle, WA`); others use only a generic label.
- **No structured address fields** — street, city, region, and postal code are not separated into distinct columns.

---

### 19. `size_or_weight`
**Purpose:** Physical dimensions or weight of the product

**Issues:**
- **Multiple units used for the same product** — the same SKU is recorded with entirely different measurement units across rows:
  - `SKU-C-1018` (CloudSleeve): `11"`, `one size`, `38cm`, `272 g`, `11 oz` all appear
- **Mixed size and weight in the same column** — products measured by inches in one row are measured by grams in another.
- **Inconsistent unit notation:**
  - Inches: `11"` / `11 inches` / `11 in` / `11 inch`
  - Ounces: `11 oz` / `11 ounces`
  - Grams: `272g` / `272 g` / `272 grams`
  - Kilograms: `0.30kg` / `0.30 kg` / `0.30 kilograms`
  - Pounds: `0.6 lbs` / `0.6 lb` / `0.6 pound`
  - Centimetres: `28cm` / `28 cm` / `28 centimetres`
- **Blank values** — many rows are missing this field entirely.

---

### 20. `return_flag`
**Purpose:** Indicates whether the item was returned

**Issues:**
- **Many blank values** — LN-004, LN-012, LN-014, LN-020, LN-022, LN-032, and many others have no value recorded.
- **Ambiguity of blank** — it is unclear whether a blank means `N` (not returned), `Unknown`, or simply was not recorded. Without a data dictionary, blank cannot safely be treated as `N`.
- A consistent policy of recording `N` when no return occurred would eliminate this ambiguity.

---

### 21. `notes`
**Purpose:** Free-text notes about the order

**Issues:**
- **Uncontrolled vocabulary** — no defined set of allowed values; the three most common notes (`gift receipt`, `late ship`, `manual discount ok`) are written consistently but there is no enforcement.
- **New value introduced mid-dataset** — `student promo` begins appearing in later rows (LN-110, LN-122, LN-127, LN-164) and was not used in earlier rows, suggesting evolving informal conventions.
- **Single-value constraint** — some orders may warrant multiple notes (e.g., both `gift receipt` and `late ship`), but only one value fits per row.
- **Blank values** — many rows have no note, which appears intentional but is inconsistent with rows where `0` discount is explicitly noted as `manual discount ok`.


## `Product_Supplier_Master`

| # | Column(s) | Issue | Rows Affected |
|---|-----------|-------|---------------|
| 11 | `sku` | Mixed casing: `"SKU-C-1002"` and `"sku-c-1002"` treated as different records | ~30 rows |
| 12 | `sku` | 18 SKUs appear 2–5× as duplicate rows; some are variants, others appear to be data entry errors | 18 SKUs |
| 13 | `category` | No controlled vocabulary: `"Tech & Student"`, `"Tech / Student"`, `"Accessories / Accessories"`, `"Student and apparels"`, comma delimiters mixed with slash | 60 |
| 14 | `cost`, `list_price` | Some values have currency codes (`"USD 6.25"`), others are bare numbers (`31.4`) — no separate currency column | 60 |
| 15 | `weight` | 10+ unit variants: `oz`, `ounces`, `g`, `grams`, `lb`, `lbs`, `pound`, `pounds`, `kg`, `kilograms` — no standard unit enforced | 49 non-null |
| 16 | `length` | 18 format variants: `"10.2 in"`, `"10.2\""`, `"10.2 inches"`, `"25.4cm"`, `"26 centimetres"` — needs numeric + unit split | 47 non-null |
| 17 | `discontinued` | Boolean Y/N field has 17 nulls (28%) — product status unknown for over a quarter of SKUs | 17 nulls |
| 18 | `category` (cross-sheet) | Category values don't align between `Sales_Dump` and `Product_Supplier_Master` — joins will produce mismatches | Both sheets |


## SQL Code used to clean Sales_Dump 

SQL
```sql
UPDATE Sales_Dump
SET sale_date = CASE 
    WHEN sale_date LIKE '%-%' THEN STR_TO_DATE(sale_date, '%m-%d-%Y')
    WHEN sale_date LIKE '%Oct%' THEN '2025-10-17' -- Handled unique natural language cases
    ELSE sale_date 
END;
2. Customer Attribute Decomposition
Issue: The customer_info column contained three distinct data points—Name, Type, and Loyalty status—concatenated into a single string.
Solution: Employed SUBSTRING_INDEX and TRIM to split the string into atomic columns, following the principle of Database Normalization.

SQL
-- Extracting Name
UPDATE Sales_Dump SET customer_name = TRIM(SUBSTRING_INDEX(customer_info, ',', 1));

-- Extracting Loyalty Member Status (Boolean Conversion)
UPDATE Sales_Dump 
SET is_loyalty_member = CASE 
    WHEN customer_info LIKE '%Member%' THEN 1 
    ELSE 0 
END;
3. Geographic & Null Remediation
Issue: Missing data in shipping columns and placeholder text like "Same as billing" rendered regional reports inaccurate.
Solution: Used UPDATE statements to map billing data to shipping fields where necessary and parsed combined city/region strings.

SQL
UPDATE Sales_Dump 
SET ship_city = TRIM(SUBSTRING_INDEX(location_string, ',', 1)),
    ship_region = TRIM(SUBSTRING_INDEX(location_string, ',', -1))
WHERE location_string IS NOT NULL;
4. Financial & Numeric Conversion
Issue: Unit prices and totals included currency symbols ($, %) and were stored as text, which blocked mathematical operations like SUM() and AVG().
Solution: Stripped non-numeric symbols using REPLACE and cast the remaining strings to the DECIMAL data type.

SQL
UPDATE Sales_Dump 
SET unit_price = CAST(REPLACE(unit_price, '$', '') AS DECIMAL(10,2)),
    line_total = CAST(REPLACE(line_total, '$', '') AS DECIMAL(10,2));
5. Referential Consistency (Normalization)
Issue: Inconsistent casing (e.g., "visa" vs. "VISA" or "sku-1" vs. "SKU-1") would cause foreign key constraint violations.
Solution: Applied UPPER and TRIM to all primary and foreign key columns to ensure perfect matching during table joins.

SQL
UPDATE Sales_Dump 
SET sku = UPPER(TRIM(sku)),
    payment_method = UPPER(TRIM(payment_method));
6. Feature Engineering (Boolean Flags)
Issue: Operational insights (e.g., late shipments or gift orders) were buried in unstructured text within the notes column.
Solution: Created new Boolean columns and used the LIKE operator to extract specific flags for advanced business intelligence.

SQL
-- Identifying Late Shipments
UPDATE Sales_Dump 
SET is_late_ship = 1 
WHERE notes LIKE '%late%';

-- Flagging Manual Discounts for Audit
UPDATE Sales_Dump 
SET is_manual_discount = 1 
WHERE notes LIKE '%promo%' OR notes LIKE '%discount%';
7. Handling "Orphan" Records
Issue: Sales records with missing customer emails or non-existent SKUs would fail to import into a strict relational schema.
Solution: Identified these records and mapped them to a pre-defined "Unknown" placeholder in the parent tables to preserve transaction volume while maintaining integrity.
```

SQL
-- Redirecting missing links to a Guest placeholder
INSERT INTO Orders (customer_id, ...)
SELECT IFNULL(c.customer_id, 9999), ...
FROM Sales_Dump s
LEFT JOIN Customers c ON s.customer_email = c.email;
Summary: Through these programmatic updates, the dataset was transformed from a low-integrity "flat" format into a clean, normalized structure ready for complex SQL joins and multi-dimensional reporting.

## SQL codes to clean the Product_Supplier_Master table
```sql
-- SKU-related cleanup
UPDATE Product_Supplier_Master
SET sku = NULLIF(UPPER(TRIM(sku)), '');

UPDATE Product_Supplier_Master
SET alt_sku = NULLIF(UPPER(TRIM(alt_sku)), '');

UPDATE Product_Supplier_Master
SET parent_sku = NULLIF(UPPER(TRIM(parent_sku)), '');

-- Product description cleanup
UPDATE Product_Supplier_Master
SET product_description = NULLIF(TRIM(product_description), '');

-- Category standardization
UPDATE Product_Supplier_Master
SET category =
    CASE
        WHEN category LIKE '%Tech%' THEN 'Tech'
        WHEN category LIKE '%Accessories%' THEN 'Accessories'
        WHEN category LIKE '%Apparel%' THEN 'Apparel'
        WHEN category LIKE '%Audio%' THEN 'Audio'
        WHEN category LIKE '%Lifestyle%' THEN 'Lifestyle'
        WHEN category LIKE '%School%' THEN 'School'
        ELSE category
    END;

-- Vendor name cleanup
UPDATE Product_Supplier_Master
SET vendor_name = 'Urban Source'
WHERE vendor_name = 'Urban Sources';

-- Vendor phone cleanup
UPDATE Product_Supplier_Master
SET vendor_phone =
    REPLACE(
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(vendor_phone, '-', ''),
                '(', ''),
            ')', ''),
        ' ', ''),
    '.', '');

-- Vendor representative cleanup
UPDATE Product_Supplier_Master
SET vendor_rep = NULLIF(TRIM(vendor_rep), '');

UPDATE Product_Supplier_Master
SET vendor_rep = TRIM(SUBSTRING_INDEX(vendor_rep, '/', 1));

UPDATE Product_Supplier_Master
SET vendor_rep = 'Anika Roy'
WHERE vendor_rep = 'Ms. Anika Roy';

UPDATE Product_Supplier_Master
SET vendor_rep = 'Mia Diaz'
WHERE vendor_rep LIKE 'Mia Dia%';

-- Cost cleanup
UPDATE Product_Supplier_Master
SET cost = TRIM(
    REPLACE(
        REPLACE(cost, 'CAD ', ''),
        'USD ', ''
    )
);

UPDATE Product_Supplier_Master
SET cost = ROUND(CAST(cost AS DECIMAL(10,2)), 2);

-- List price cleanup
UPDATE Product_Supplier_Master
SET list_price = TRIM(
    REPLACE(
        REPLACE(list_price, 'CAD ', ''),
        'USD ', ''
    )
);

UPDATE Product_Supplier_Master
SET list_price = ROUND(CAST(list_price AS DECIMAL(10,2)), 2);

-- Reorder level cleanup
UPDATE Product_Supplier_Master
SET reorder_level = '10'
WHERE LOWER(TRIM(reorder_level)) LIKE '%ten%';

UPDATE Product_Supplier_Master
SET reorder_level = CAST(reorder_level AS UNSIGNED);

-- Pack size cleanup
UPDATE Product_Supplier_Master
SET pack_size =
    CASE
        WHEN pack_size IN ('1', '1 each') THEN '1 each'
        WHEN pack_size IN ('2/pack', '2-pack', '2 pack') THEN '2-pack'
        WHEN pack_size IN ('3-pack', '3 pack') THEN '3-pack'
        WHEN LOWER(pack_size) LIKE '%case of 6%' THEN 'Case of 6'
        ELSE pack_size
    END;

-- Weight cleanup and standardization to grams
UPDATE Product_Supplier_Master
SET weight = NULL
WHERE LOWER(TRIM(weight)) = 'na';

UPDATE Product_Supplier_Master
SET weight = REPLACE(weight, ' ', '');

UPDATE Product_Supplier_Master
SET weight =
    CASE
        WHEN weight LIKE '%g' AND weight NOT LIKE '%kg'
            THEN CAST(REPLACE(REPLACE(weight, 'grams', ''), 'g', '') AS DECIMAL(10,2))
        WHEN weight LIKE '%kg'
            THEN CAST(REPLACE(weight, 'kg', '') AS DECIMAL(10,2)) * 1000
        WHEN weight LIKE '%oz'
            THEN CAST(REPLACE(weight, 'oz', '') AS DECIMAL(10,2)) * 28.3495
        WHEN weight LIKE '%lb%'
            THEN CAST(REPLACE(REPLACE(weight, 'lb', ''), 'pound', '') AS DECIMAL(10,2)) * 453.592
        ELSE NULL
    END;

UPDATE Product_Supplier_Master
SET weight = ROUND(CAST(weight AS DECIMAL(10,2)), 2);

-- Length cleanup and standardization to centimeters
UPDATE Product_Supplier_Master
SET length = REPLACE(length, ' ', '');

UPDATE Product_Supplier_Master
SET length =
    CASE
        WHEN LOWER(length) LIKE '%cm'
            THEN CAST(REPLACE(LOWER(length), 'cm', '') AS DECIMAL(10,2))
        WHEN LOWER(length) LIKE '%in'
            THEN CAST(REPLACE(LOWER(length), 'in', '') AS DECIMAL(10,2)) * 2.54
        WHEN length LIKE '%"'
            THEN CAST(REPLACE(length, '"', '') AS DECIMAL(10,2)) * 2.54
        ELSE NULL
    END;

UPDATE Product_Supplier_Master
SET length = ROUND(CAST(length AS DECIMAL(10,2)), 2);

-- Discontinued cleanup
UPDATE Product_Supplier_Master
SET discontinued =
    CASE
        WHEN UPPER(TRIM(discontinued)) IN ('Y', 'YES', 'TRUE', '1') THEN 'Y'
        WHEN UPPER(TRIM(discontinued)) IN ('N', 'NO', 'FALSE', '0') THEN 'N'
        ELSE NULL
    END;

-- Notes cleanup
UPDATE Product_Supplier_Master
SET notes = NULLIF(TRIM(notes), '');

-- Redundant column removal
ALTER TABLE Product_Supplier_Master DROP COLUMN price_currency;
ALTER TABLE Product_Supplier_Master DROP COLUMN price_numeric;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_value;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_unit;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_kg;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_value;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_unit;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_cm;

-- Parent SKU validation
SELECT DISTINCT p.parent_sku
FROM Product_Supplier_Master p
LEFT JOIN Product_Supplier_Master c
    ON p.parent_sku = c.sku
WHERE p.parent_sku IS NOT NULL
  AND c.sku IS NULL;
```

## **Justification for Cleaning Steps**
---

## Sales_Dump Cleaning

### Dates
- Standardized to ISO format (YYYY-MM-DD)
- CORD → DD-MM-YYYY
- UORD → MM-DD-YYYY
- Manual corrections applied where necessary

### Country
- Normalized to US / CA
- Filled nulls using order_id prefix logic

### Payment Method
- Converted to title case
- Standardized abbreviations (MC → Mastercard)

### Discounts
- Converted all values to decimals (e.g., 10% → 0.10)
- Extracted numeric values from text

### Tax
- Converted to decimal format
- Filled using order-level consistency
- Remaining nulls left unchanged

### Prices / Line Totals
- Removed currency symbols
- Stored as numeric
- Left null where inconsistent

### Quantity
- Extracted numeric values from text
- Stored as integer

### Customer Info
- Split into first name, last name, type
- Defaulted missing types to Standard

### Category
- Standardized into:
  - Tech, Apparel, Audio, School, Accessories, Lifestyle, Desk Setup

### Size / Weight
- Converted:
  - Weight → grams
  - Length → cm
- Preserved "one size"

### Product Descriptions
- Converted ALL CAPS to title case
- Corrected mismatches using reference table

### Emails
- Fixed malformed entries
- Filled using composite key where unique
- Remaining nulls left unchanged
---

## Product_Supplier_Master Cleaning

### Category
- Standardized into 7 categories

### Cost / List Price
- Removed currency prefixes
- Converted to numeric

### Weight / Length
- Converted to:
  - weight_g
  - length_cm

### Duplicates
- Removed exact duplicates
- Preserved SKU variants

### Reorder Level
- Converted text values to numeric
- Corrected inconsistencies

### Parent SKU
- Filled for variants
- Linked to base product

### Vendor Data
- Standardized names and phone numbers
- Split representative names into components

### Pack Size
- Standardized formatting
  
### Identifier Standardization (SKU Integrity)
- Trimmed whitespace and standardized all SKU, alt_sku, and parent_sku values
- Converted all identifiers to uppercase
- Replaced blanks with NULL
- Validated that all parent_sku values exist in sku to preserve referential integrity

### Vendor Data Standardization
- Removed inconsistencies in vendor naming (singular vs plural)
- Standardized phone numbers to numeric format
- Cleaned representative names (removed titles, extra notes, corrected typos)

### Removed Redundant Columns
- Dropped helper columns (price_currency, weight_unit, length_unit, etc.)
- Reduced redundancy and improved normalization

### Discontinued Flag
- Standardized to Y/N format for consistency

### Notes Field
- Preserved as free text
- Trimmed whitespace but avoided heavy transformation to retain meaning

### Data Integrity Validation
- Confirmed all parent-child SKU relationships are valid
- Ensured cleaned dataset aligns with normalized database structure


## **Queries**

**1. Top Products by Revenue and Country**
Natural Language Question: Which products generated the highest total sales revenue, by country?

```sql
SELECT ship_country, sku, product_description, SUM(line_total) AS total_revenue
FROM Sales_Dump
GROUP BY ship_country, sku, product_description
ORDER BY ship_country, total_revenue DESC;
```
<img width="1470" height="719" alt="image" src="https://github.com/user-attachments/assets/b0c670c5-d391-4bfc-9267-ac41d46cdcd0" />



**2. ‎Employee Performance vs. Managerial Peer Average**
Natural Language Question: Which employees handled the largest number of orders, and how do their results compare with other employees under the same manager?

```sql
SELECT employee_ref, manager_ref, COUNT(DISTINCT order_id) AS order_count,
       AVG(COUNT(DISTINCT order_id)) OVER (PARTITION BY manager_ref) AS peer_avg_orders
FROM Sales_Dump
GROUP BY employee_ref, manager_ref
ORDER BY order_count DESC;
```
<img width="1444" height="689" alt="image" src="https://github.com/user-attachments/assets/dff4b031-a3a6-4740-91d6-4a88927fa467" />



**3. Multi-Category Vendors**
Natural Language Question: Which vendors supply products that appear in more than one category?

```sql
SELECT vendor_name, COUNT(DISTINCT category) AS category_count
FROM Product_Supplier_Master
GROUP BY vendor_name
HAVING COUNT(DISTINCT category) > 1;
```

<img width="1430" height="574" alt="image" src="https://github.com/user-attachments/assets/2570f1fc-c862-4157-81e7-569e8c92c14b" />



**4. Student Demographic Sales Impact**
Natural Language Question: What is the total revenue and average line total for orders associated with "Student" customers?
Business Justification: Since Northline focuses on student-friendly gear, this query quantifies the actual financial impact of the student demographic. If the average spend is lower than guests, management might implement "Student Bundle" deals to increase order value.

```sql
SELECT COUNT(line_id) AS total_student_lines, 
       SUM(line_total) AS total_student_revenue,
       AVG(line_total) AS avg_student_line_value
FROM Sales_Dump
WHERE customer_info LIKE '%Student%';
```



**5. High-Risk Inventory Audit (Returns vs. Sales)**
Natural Language Question: Which products have a return rate higher than 10% and what is the total revenue lost from those returns?
Business Justification: High return rates usually signal quality issues or misleading descriptions. By identifying these "High-Risk" items, the data wrangler can pinpoint specific vendors or products that are causing high processing costs and lost margins.

```sql
SELECT sku, product_description,
       (COUNT(CASE WHEN return_flag = 'Y' THEN 1 END) / COUNT(*)) * 100 AS return_rate,
       SUM(CASE WHEN return_flag = 'Y' THEN line_total ELSE 0 END) AS revenue_lost
FROM Sales_Dump
GROUP BY sku, product_description
HAVING return_rate > 10
ORDER BY revenue_lost DESC;
```
<img width="1435" height="724" alt="image" src="https://github.com/user-attachments/assets/32361390-7ed5-4cec-8c71-15c1faf27ede" />



**6. Monthly Sales Growth by Category**
Natural Language Question: What is the average discount given per category in the United States versus Canada?
Business Justification: This identifies if one region is receiving significantly more discounts to move inventory. If Canadian orders have higher discounts, it may indicate a need to adjust base pricing in that country to maintain a healthy profit margin.

```sql
SELECT ship_country, category, 
       AVG(CASE 
           WHEN discount LIKE '%%' THEN CAST(REPLACE(discount, '%', '') AS DECIMAL) / 100 
           ELSE CAST(discount AS DECIMAL) 
       END) AS avg_discount_value
FROM Sales_Dump
GROUP BY ship_country, category
ORDER BY ship_country, avg_discount_value DESC;
```
<img width="1472" height="685" alt="image" src="https://github.com/user-attachments/assets/c8fe5c0c-81cd-4175-87f7-25e133482a8e" />

    
