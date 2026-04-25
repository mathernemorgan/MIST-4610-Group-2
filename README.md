**Team Members / Roles**

Hiya Shah (Data Wrangler), Morgan Matherne (Database Designer), Mark Monzer (Conceptual Modeler), Roshan Gadiraju (SQL Writer), Zeynep Koseoglu (Group Leader)

**Conceptual Model / Database Description**

Our database was designed to transform the two messy source spreadsheets, Sales_Dump and Product_Supplier_Master, into a compact relational model that supports data cleaning, structured storage, and the required SQL analysis. This matches the project goal of creating a conceptual model, implementing a database, and using that database to answer business questions. The assignment also emphasizes that a strong final model should remain relatively compact, with about 8 to 10 entities, and should clearly show entities, attributes, identifiers, and relationships.

The model centers on the retail sales process for Northline Outfitters, a small online retailer that purchases products from outside vendors and sells them directly to customers in the United States and Canada. The database separates operational data into core business entities so that customer, employee, product, vendor, order, and payment information are not stored repeatedly in a single spreadsheet row. This reduces redundancy and makes the data easier to query and maintain. The design is intended to reflect the business case described in the project instructions, where sales records contain order-level and line-level information and product records contain vendor, category, and inventory-related details.

The main transactional entities in the model are Orders and OrderLines. The Orders table stores order-level information such as the order identifier, sale date, shipping location, customer, employee, payment method, and country. The OrderLines table stores the individual product lines associated with each order, including quantity, unit price, discount amount, tax amount, line total, return status, and any size or weight detail captured on the sales record. This separation is important because one order can contain multiple products, so storing everything in a single order table would create repeating groups and violate normalization principles. This structure also directly supports the project’s required revenue and sales-performance queries, which depend on line-level sales data and order-level shipping context.

The product side of the model is built around the Product entity. Each product is identified by a SKU and stores attributes such as alternate SKU, product description, list price, discontinuation status, weight, length, reorder level, pack size, notes, and category. The inclusion of parent_sku supports product variants by allowing one product to reference a parent product family. This reflects the source spreadsheet, which explicitly included alternate SKU values and parent SKU values for variant or related products. Product data was separated from sales transactions so that product attributes are maintained once and linked to each order line through the SKU relationship.

Supporting the Product entity are the Categories, Vendor, and product_vendor tables. Categories stores the standardized product categories used in the cleaned data model, while Vendor stores supplier information such as vendor name, phone number, and vendor representative. The product_vendor bridge table represents the relationship between products and vendors and stores vendor-specific cost data. This design allows the database to represent situations in which the same product may be associated with a vendor record and helps support the required query asking which vendors supply products that appear in more than one category. Even if many products currently map to one primary vendor, using a bridge table makes the design more flexible and better aligned with the business context of supplier-managed inventory.

The people-related side of the model includes Customer and Employees. The Customer table stores customer identifiers, names, email addresses, customer type, notes, and loyalty information. This structure was created because the source data embedded customer details inside a single customer_info field and also included email separately, so separating customer information into its own table reduces duplication across orders. The Employees table stores employee identifiers and manager information so that employee performance can be analyzed both individually and relative to employees under the same manager. This directly supports the required query comparing employees’ order-handling results within managerial groups.

The PaymentMethods table standardizes how payment types are stored. Instead of repeating raw payment text such as Visa, Mastercard, or Debit on every order row, the model stores payment methods in a reference table and links them to orders and, in your current design, order lines. This improves consistency and reduces repeated text values. It also reflects the assignment’s emphasis on cleaning inconsistent payment formats from the source spreadsheets before using the data in queries.

In terms of relationships, the most important business rules in the model are as follows. A Customer can place many Orders, but each order belongs to one customer. An Employee can handle many orders, but each order is associated with one employee. An Order can contain many OrderLines, but each order line belongs to one order. A Product can appear in many order lines, but each order line refers to one product. A Category can classify many products, while each product belongs to one category. A Vendor can be linked to many products through the product_vendor table, and a product can also be linked to one or more vendors through that same bridge table. A PaymentMethod can be used by many orders, while each order references one payment method. These relationships were chosen to make the model both normalized and practical for the required business questions.

Overall, this model is designed to be parsimonious but still expressive enough to support the retail case. It separates the original spreadsheets into distinct business entities, reduces redundancy, enforces clearer identifiers and relationships, and creates a structure that is suitable for both data quality improvement and SQL analysis. In that sense, the model follows the project guidance to keep the design compact, well-justified, and focused on useful business queries rather than over-engineering the schema.

**Data Cleaning Process: Product_Supplier_Master**

The Product_Supplier_Master dataset contained multiple data quality issues, including inconsistent identifiers, mixed text and numeric formats, embedded currency labels, inconsistent units of measure, redundant helper columns, and unstructured notes. These issues were cleaned using SQL so the data could be standardized and loaded into the final relational model. The source spreadsheet included product, vendor, pricing, packaging, measurement, discontinuation, and parent SKU fields, so the cleaning process focused on making each of those attributes consistent and usable in the database.

**The following SQL statements were used to clean the Product_Supplier_Master table, along with the justification for each step.**

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

**Justification for Cleaning Steps**

SKU, alt_sku, and parent_sku were standardized by trimming whitespace, converting values to uppercase, and replacing blanks with NULL. This was necessary because SKU fields act as identifiers, and inconsistent formatting would create duplicate-looking products or broken parent-child relationships. The spreadsheet documentation makes clear that SKU is the internal product code and that parent_sku is used to link variants to a main product, so these fields had to be consistent for referential integrity.

Product descriptions were trimmed to remove unnecessary whitespace. This was a light cleanup step designed to preserve the original product names while making the text more consistent.

Category values were standardized into a smaller set of single, primary categories. This was necessary because the raw data included inconsistent and multi-valued entries, while the source description shows that category is supposed to represent the product’s category such as Tech, Accessories, Apparel, Lifestyle, School, or Audio. Standardizing these values improves grouping, reporting, and normalization.

Vendor names were cleaned to eliminate naming inconsistencies, such as singular versus plural forms of the same supplier. Since vendor information is meant to identify suppliers, inconsistent names would produce duplicate vendor records in the final design. The file description also indicates that vendor name, phone, and representative belong together as supplier information.

Vendor phone numbers were standardized by removing punctuation and spaces so that all values followed one consistent numeric format. This improves consistency and makes it easier to compare and validate phone values.

Vendor representatives were cleaned by trimming whitespace, removing extra slash-delimited notes, removing titles, and correcting typographical inconsistencies. This was necessary so that the same representative would not appear as multiple slightly different values.

Cost and list price were cleaned by removing embedded currency text and converting the results into numeric values with two decimal places. The source spreadsheet defines these fields as monetary values, so leaving currency labels in the data would prevent proper calculations and comparisons. Standardizing them made the values usable for price, margin, and vendor analyses.

Reorder level was cleaned by converting text values such as “ten” into numeric form. This was necessary because reorder level is intended to be a minimum stock threshold, which should be stored as a number rather than text.

Pack size was standardized so that equivalent values were represented consistently. Since this field describes how products are packaged or counted for ordering, normalizing the formats improves readability and reduces duplicate categorical values.

Weight values were cleaned by removing invalid entries such as “NA,” standardizing formatting, converting all measurements into grams, and rounding to two decimal places. This step was required because the project instructions explicitly note that the source spreadsheets may contain both metric and imperial measurements. Standardizing all weights into one unit made the data comparable across products.

Length values were similarly standardized by converting all measurements into centimeters and rounding to two decimal places. This was necessary because length values also appeared in mixed measurement systems and inconsistent text formats.

Discontinued was standardized into a consistent Y/N format. The source description identifies this column as a field indicating whether a product has been discontinued, so a consistent binary representation makes the attribute easier to query and interpret.

Notes were lightly cleaned by trimming whitespace, but the field was retained as free text. The spreadsheet description states that notes contain extra comments such as family variant notes or seasonal notes, so the field was preserved as supplementary information rather than heavily normalized.

Redundant helper columns such as price_currency, price_numeric, weight_value, weight_unit, weight_kg, length_value, length_unit, and length_cm were removed because they were either empty, incomplete, duplicated other information, or became unnecessary after standardization. Removing them reduced redundancy and made the final table simpler and more consistent.

Parent SKU validation was performed to confirm that every non-null parent_sku matched an existing sku. This step ensured that product family relationships remained valid after the identifier cleanup and supported the final normalized product design.

Overall, these cleaning steps transformed Product_Supplier_Master from a messy spreadsheet export into a structured dataset suitable for loading into normalized entities such as Product, Vendor, and Category. This directly supports the project requirement to identify major data quality issues, explain how they were resolved, and include SQL statements used to standardize, split, convert, or update the imported data.
