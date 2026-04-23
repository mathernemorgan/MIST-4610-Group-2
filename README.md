# MIST-4610-Group-2

**Team Members:** Hiya Shah, 

**🧹 Data Cleaning SQL Queries**

**🔑 SKU Standardization**
UPDATE Product_Supplier_Master
SET sku = NULLIF(UPPER(TRIM(sku)), '');

UPDATE Product_Supplier_Master
SET alt_sku = NULLIF(UPPER(TRIM(alt_sku)), '');

UPDATE Product_Supplier_Master
SET parent_sku = NULLIF(UPPER(TRIM(parent_sku)), '');

**📝 Product Description**
