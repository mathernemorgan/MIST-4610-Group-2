

**🧹 Data Cleaning SQL Queries**

**🔑 SKU Standardization**

UPDATE Product_Supplier_Master
SET sku = NULLIF(UPPER(TRIM(sku)), '');

UPDATE Product_Supplier_Master
SET alt_sku = NULLIF(UPPER(TRIM(alt_sku)), '');

UPDATE Product_Supplier_Master
SET parent_sku = NULLIF(UPPER(TRIM(parent_sku)), '');
📝 Product Description
UPDATE Product_Supplier_Master
SET product_description = NULLIF(TRIM(product_description), '');
🗂️ Category Standardization
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
    
**🏢 Vendor Name Cleanup**

UPDATE Product_Supplier_Master
SET vendor_name = 'Urban Source'
WHERE vendor_name = 'Urban Sources';
📞 Vendor Phone Standardization
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
    
**👤 Vendor Representative Cleanup**

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

**💰 Cost Cleaning**

UPDATE Product_Supplier_Master
SET cost = TRIM(
    REPLACE(
        REPLACE(cost, 'CAD ', ''),
        'USD ', ''
    )
);

UPDATE Product_Supplier_Master
SET cost = ROUND(CAST(cost AS DECIMAL(10,2)), 2);

**💰 List Price Cleaning**

UPDATE Product_Supplier_Master
SET list_price = TRIM(
    REPLACE(
        REPLACE(list_price, 'CAD ', ''),
        'USD ', ''
    )
);

UPDATE Product_Supplier_Master
SET list_price = ROUND(CAST(list_price AS DECIMAL(10,2)), 2);

**📦 Reorder Level Fix**

UPDATE Product_Supplier_Master
SET reorder_level = '10'
WHERE LOWER(TRIM(reorder_level)) LIKE '%ten%';

UPDATE Product_Supplier_Master
SET reorder_level = CAST(reorder_level AS UNSIGNED);

**📦 Pack Size Standardization**

UPDATE Product_Supplier_Master
SET pack_size =
    CASE
        WHEN pack_size IN ('1', '1 each') THEN '1 each'
        WHEN pack_size IN ('2/pack', '2-pack', '2 pack') THEN '2-pack'
        WHEN pack_size IN ('3-pack', '3 pack') THEN '3-pack'
        WHEN LOWER(pack_size) LIKE '%case of 6%' THEN 'Case of 6'
        ELSE pack_size
    END;
    
**⚖️ Weight Standardization (→ grams)**

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

**📏 Length Standardization (→ cm)**

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

**🔁 Boolean Cleanup (Discontinued)**

UPDATE Product_Supplier_Master
SET discontinued =
    CASE
        WHEN UPPER(TRIM(discontinued)) IN ('Y','YES','TRUE','1') THEN 'Y'
        WHEN UPPER(TRIM(discontinued)) IN ('N','NO','FALSE','0') THEN 'N'
        ELSE NULL
    END;
    
**📝 Notes Cleanup (light only)**

UPDATE Product_Supplier_Master
SET notes = NULLIF(TRIM(notes), '');

**🧱 Remove Redundant Columns**

ALTER TABLE Product_Supplier_Master DROP COLUMN price_currency;
ALTER TABLE Product_Supplier_Master DROP COLUMN price_numeric;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_value;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_unit;
ALTER TABLE Product_Supplier_Master DROP COLUMN weight_kg;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_value;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_unit;
ALTER TABLE Product_Supplier_Master DROP COLUMN length_cm;
🔗 Data Integrity Check (parent_sku)
SELECT DISTINCT p.parent_sku
FROM Product_Supplier_Master p
LEFT JOIN Product_Supplier_Master c
    ON p.parent_sku = c.sku
WHERE p.parent_sku IS NOT NULL
  AND c.sku IS NULL;

****  Data Cleaning Justification********

Each data cleaning step was performed to address specific data quality issues and ensure the dataset was consistent, reliable, and suitable for relational modeling.

SKU Standardization:
SKU-related fields (sku, alt_sku, parent_sku) were standardized by trimming whitespace, converting values to uppercase, and replacing blanks with NULL. This was necessary to ensure consistency in product identifiers and to prevent duplicate records caused by formatting differences (e.g., “sku-1001” vs. “SKU-1001”). Consistent identifiers are critical for joins and maintaining referential integrity.

Product Description Cleanup:
Product descriptions were trimmed to remove unnecessary whitespace. This ensured consistency in text fields while preserving the original meaning, since descriptions are free-text and should not be over-normalized.

Category Standardization:
Categories were normalized by removing multi-valued entries and mapping them to a single primary category. This was necessary because inconsistent values (e.g., “Tech / Student”) prevent accurate grouping and violate normalization principles. Assigning each product to one category improves query performance and analytical clarity.

Vendor Name Cleanup:
Vendor names were standardized to remove inconsistencies (e.g., “Urban Sources” → “Urban Source”). This prevents duplicate vendor records and ensures accurate relationships between products and vendors in the final schema.

Vendor Phone Standardization:
Formatting characters were removed from phone numbers to ensure a consistent numeric format. This improves data uniformity and allows for easier comparison, validation, and potential integration with external systems.

Vendor Representative Cleanup:
Vendor representative names were cleaned by removing titles, correcting typographical errors, and eliminating extra notes. This ensures consistent naming and avoids duplication caused by minor variations.

Price Cleaning (Cost and List Price):
Currency indicators (USD, CAD, $) were removed from price fields, and values were converted to numeric format with consistent decimal precision. This was necessary because embedded text prevents mathematical operations and accurate financial analysis.

Reorder Level Standardization:
Text-based values (e.g., “ten”) were converted into numeric form. This ensures the field can be used for calculations and inventory analysis.

Pack Size Standardization:
Different representations of packaging (e.g., “2/pack,” “2 pack”) were standardized into consistent formats. This improves readability and prevents duplicate categorical values.

Weight Standardization:
Weight values were converted from multiple units (kg, lb, oz) into a single unit (grams). This ensures consistency and allows for accurate comparisons and calculations across products.

Length Standardization:
Length values were converted into centimeters from mixed units such as inches. Standardizing units eliminates ambiguity and supports consistent analysis.

Discontinued Field Standardization:
The discontinued field was standardized into a consistent Y/N format. This ensures clarity and allows the field to function as a proper binary attribute.

Removal of Redundant Columns:
Columns such as price_currency, price_numeric, weight_value, weight_unit, weight_kg, length_value, length_unit, and length_cm were removed because they were either empty, duplicated information, or became unnecessary after standardization. Removing these columns simplifies the dataset and supports normalization.

Notes Field Handling:
The notes column was retained but only lightly cleaned by trimming whitespace. It was not normalized because it contains unstructured, free-text information that does not fit well into a structured schema.

Parent SKU Validation:
Parent SKU relationships were validated to ensure referential integrity. Ensuring that each parent_sku corresponds to an existing sku prevents broken relationships and maintains data consistency.
