---
name: magento-declarative-schema-manager
description: Transitions traditional SQL table management to declarative db_schema.xml standard, enabling validation and declarative data migration. Use when creating database tables, upgrading schemas, or modernizing legacy install/upgrade scripts. Requires Magento 2.3+ knowledge.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Magento Declarative Schema Manager

Converts traditional InstallSchema/UpgradeSchema scripts to declarative `db_schema.xml` format introduced in Magento 2.3, providing automated schema validation, rollback capability, and cleaner database management.

## When to Use

- Creating new database tables (Magento 2.3+)
- Upgrading existing install/upgrade scripts
- Modernizing legacy modules
- Managing database schema changes
- Adding indexes, foreign keys, constraints
- Database migrations

## Benefits of Declarative Schema

- ✅ No manual SQL scripts
- ✅ Automatic validation
- ✅ Rollback support (dry-run mode)
- ✅ Whitelist columns for data persistence
- ✅ Easier to maintain and read
- ✅ Framework handles differences automatically

## File Locations

- **Schema definition:** `etc/db_schema.xml`
- **Schema whitelist:** `etc/db_schema_whitelist.json`

## Execution Workflow

1. **Analyze requirements**
   - Identify tables, columns needed
   - Define indexes, foreign keys
   - Plan data types and constraints

2. **Create db_schema.xml**
   - Define table structure
   - Add columns with proper types
   - Define constraints and indexes
   - Add foreign keys

3. **Generate whitelist**
   - Run `setup:db-declaration:generate-whitelist`
   - Validates schema correctness

4. **Test migration**
   - Use dry-run mode first
   - Apply changes
   - Verify in database

5. **Data patches (if needed)**
   - Create DataPatch for initial data
   - Implement apply() method

## db_schema.xml Structure

### Complete Example

**File:** `app/code/Vendor/Module/etc/db_schema.xml`
```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    
    <!-- Define table -->
    <table name="vendor_module_rating" resource="default" engine="innodb" comment="Product Ratings">
        
        <!-- Primary key -->
        <column xsi:type="int" name="rating_id" unsigned="true" nullable="false" identity="true" comment="Rating ID"/>
        
        <!-- Foreign key column -->
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="false" comment="Customer ID"/>
        
        <!-- Regular columns -->
        <column xsi:type="varchar" name="product_sku" nullable="false" length="64" comment="Product SKU"/>
        <column xsi:type="smallint" name="rating" unsigned="true" nullable="false" comment="Rating Value"/>
        <column xsi:type="text" name="comment" nullable="true" comment="Rating Comment"/>
        <column xsi:type="timestamp" name="created_at" on_update="false" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" on_update="true" nullable="false" default="CURRENT_TIMESTAMP" comment="Updated At"/>
        
        <!-- Primary key constraint -->
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="rating_id"/>
        </constraint>
        
        <!-- Foreign key constraint -->
        <constraint xsi:type="foreign" 
                    referenceId="VENDOR_MODULE_RATING_CUSTOMER_ID_CUSTOMER_ENTITY_ENTITY_ID" 
                    table="vendor_module_rating" 
                    column="customer_id" 
                    referenceTable="customer_entity" 
                    referenceColumn="entity_id" 
                    onDelete="CASCADE"/>
        
        <!-- Unique constraint -->
        <constraint xsi:type="unique" referenceId="VENDOR_MODULE_RATING_CUSTOMER_ID_PRODUCT_SKU">
            <column name="customer_id"/>
            <column name="product_SKU"/>
        </constraint>
        
        <!-- Indexes -->
        <index referenceId="VENDOR_MODULE_RATING_PRODUCT_SKU" indexType="btree">
            <column name="product_sku"/>
        </index>
        
        <index referenceId="VENDOR_MODULE_RATING_CUSTOMER_ID" indexType="btree">
            <column name="customer_id"/>
        </index>
        
        <index referenceId="VENDOR_MODULE_RATING_RATING" indexType="btree">
            <column name="rating"/>
        </index>
        
    </table>
</schema>
```

## Column Types

| XSI Type | Database Type | Usage |
|----------|---------------|-------|
| `int` | INT | Integer values |
| `smallint` | SMALLINT | Small integer (0-65535) |
| `bigint` | BIGINT | Large integer |
| `varchar` | VARCHAR | String with max length |
| `text` | TEXT | Long text |
| `decimal` | DECIMAL | Decimal numbers |
| `boolean` | TINYINT(1) | True/false |
| `date` | DATE | Date only |
| `datetime` | DATETIME | Date and time |
| `timestamp` | TIMESTAMP | Auto-updating timestamp |
| `blob` | BLOB | Binary data |

## Column Attributes

```xml
<column xsi:type="varchar" 
        name="product_sku"           <!-- Column name -->
        nullable="false"             <!-- Allow NULL? -->
        length="64"                  <!-- Max length (varchar, varbinary) -->
        unsigned="true"              <!-- Unsigned (numeric) -->
        identity="true"              <!-- AUTO_INCREMENT -->
        default="0"                  <!-- Default value -->
        on_update="true"             <!-- ON UPDATE CURRENT_TIMESTAMP (timestamp) -->
        comment="Product SKU"        <!-- Column comment -->
        disabled="false"             <!-- Disable column (for removal) -->
/>
```

## Constraints

### Primary Key
```xml
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="entity_id"/>
</constraint>
```

### Foreign Key
```xml
<constraint xsi:type="foreign" 
            referenceId="FK_NAME" 
            table="current_table" 
            column="foreign_key_column" 
            referenceTable="referenced_table" 
            referenceColumn="referenced_column" 
            onDelete="CASCADE|SET NULL|NO ACTION|RESTRICT"/>
```

### Unique Constraint
```xml
<constraint xsi:type="unique" referenceId="UNIQUE_CONSTRAINT_NAME">
    <column name="column1"/>
    <column name="column2"/>
</constraint>
```

## Indexes

### Simple Index
```xml
<index referenceId="INDEX_NAME" indexType="btree">
    <column name="column_name"/>
</index>
```

### Composite Index
```xml
<index referenceId="INDEX_NAME" indexType="btree">
    <column name="column1"/>
    <column name="column2"/>
</index>
```

### Full-text Index
```xml
<index referenceId="FTI_NAME" indexType="fulltext">
    <column name="searchable_field"/>
</index>
```

## Common Patterns

### Magento Standard Columns
```xml
<!-- Auto-increment ID -->
<column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Entity ID"/>

<!-- Created/Updated timestamps -->
<column xsi:type="timestamp" name="created_at" on_update="false" nullable="false" default="CURRENT_TIMESTAMP"/>
<column xsi:type="timestamp" name="updated_at" on_update="true" nullable="false" default="CURRENT_TIMESTAMP"/>

<!-- Status field -->
<column xsi:type="smallint" name="is_active" unsigned="true" nullable="false" default="1" comment="Is Active"/>
```

### EAV Attribute Table Pattern
```xml
<table name="vendor_module_entity_varchar" resource="default" engine="innodb">
    <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="smallint" name="attribute_id" unsigned="true" nullable="false" default="0"/>
    <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" default="0"/>
    <column xsi:type="smallint" name="store_id" unsigned="true" nullable="false" default="0"/>
    <column xsi:type="varchar" name="value" nullable="true" length="255"/>
    
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="value_id"/>
    </constraint>
    
    <constraint xsi:type="unique" referenceId="UNIQUE_ENTITY_ATTRIBUTE_STORE">
        <column name="entity_id"/>
        <column name="attribute_id"/>
        <column name="store_id"/>
    </constraint>
</table>
```

## Modifying Existing Tables

### Add Column
```xml
<table name="existing_table">
    <column xsi:type="varchar" name="new_column" nullable="true" length="255" comment="New Column"/>
</table>
```

### Modify Column
```xml
<table name="existing_table">
    <!-- Increase varchar length -->
    <column xsi:type="varchar" name="existing_column" nullable="false" length="512" comment="Modified Column"/>
</table>
```

### Remove Column
```xml
<table name="existing_table">
    <column xsi:type="varchar" name="column_to_remove" disabled="true"/>
</table>
```

### Drop Table
```xml
<table name="table_to_remove" disabled="true"/>
```

## Commands

### Generate Whitelist
```bash
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
```

### Dry Run (Preview Changes)
```bash
bin/magento setup:upgrade --dry-run=1
```

### Apply Schema Changes
```bash
bin/magento setup:upgrade
```

### Convert Legacy Scripts
```bash
bin/magento setup:db-declaration:generate-patch --module=Vendor_Module
```

## Data Patches

For initial data or data migrations:

**File:** `app/code/Vendor/Module/Setup/Patch/Data/AddInitialRatings.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddInitialRatings implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    
    public function __construct(ModuleDataSetupInterface $moduleDataSetup) {
        $this->moduleDataSetup = $moduleDataSetup;
    }
    
    public function apply()
    {
        $this->moduleDataSetup->getConnection()->startSetup();
        
        $table = $this->moduleDataSetup->getTable('vendor_module_rating');
        $data = [
            ['customer_id' => 1, 'product_sku' => '24-MB01', 'rating' => 5, 'comment' => 'Great!'],
            ['customer_id' => 2, 'product_sku' => '24-MB02', 'rating' => 4, 'comment' => 'Good'],
        ];
        
        $this->moduleDataSetup->getConnection()->insertMultiple($table, $data);
        
        $this->moduleDataSetup->getConnection()->endSetup();
    }
    
    public static function getDependencies()
    {
        return []; // List dependencies here
    }
    
    public function getAliases()
    {
        return [];
    }
}
```

## Migration from InstallSchema

**Before (Legacy):**
```php
<?php
namespace Vendor\Module\Setup;

use Magento\Framework\Setup\InstallSchemaInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\DB\Ddl\Table;

class InstallSchema implements InstallSchemaInterface
{
    public function install(SchemaSetupInterface $setup, ModuleContextInterface $context)
    {
        $setup->startSetup();
        
        $table = $setup->getConnection()->newTable(
            $setup->getTable('vendor_module_rating')
        )->addColumn(
            'rating_id',
            Table::TYPE_INTEGER,
            null,
            ['identity' => true, 'unsigned' => true, 'nullable' => false, 'primary' => true],
            'Rating ID'
        )->addColumn(
            'product_sku',
            Table::TYPE_TEXT,
            64,
            ['nullable' => false],
            'Product SKU'
        )->addIndex(
            $setup->getIdxName('vendor_module_rating', ['product_sku']),
            ['product_sku']
        )->setComment('Product Ratings');
        
        $setup->getConnection()->createTable($table);
        $setup->endSetup();
    }
}
```

**After (Declarative):**
```xml
<!-- etc/db_schema.xml -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_module_rating" resource="default" engine="innodb" comment="Product Ratings">
        <column xsi:type="int" name="rating_id" unsigned="true" nullable="false" identity="true" comment="Rating ID"/>
        <column xsi:type="varchar" name="product_sku" nullable="false" length="64" comment="Product SKU"/>
        
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="rating_id"/>
        </constraint>
        
        <index referenceId="VENDOR_MODULE_RATING_PRODUCT_SKU" indexType="btree">
            <column name="product_sku"/>
        </index>
    </table>
</schema>
```

## Best Practices

1. **Always generate whitelist** after schema changes
2. **Test with dry-run** before applying
3. **Use meaningful referenceIds** for constraints/indexes
4. **Follow naming conventions** (uppercase, underscores)
5. **Set appropriate column types** and lengths
6. **Add comments** to all columns and tables
7. **Use foreign keys** for referential integrity
8. **Create indexes** on frequently queried columns
9. **Use Data Patches** for initial/migration data
10. **Version control** both XML and whitelist

## Quality Bar

- Schema validates without errors
- Whitelist generated successfully
- Foreign keys reference correct tables/columns
- Indexes created on query-heavy columns
- All columns have comments
- Follows Magento naming conventions
- Dry-run shows expected changes
- Applied schema matches database
