---
name: dockergento-mysql-controller
description: Manages MySQL database operations in Dockergento using hm mysql commands. Use when importing SQL dumps, executing queries, managing databases, or troubleshooting database issues.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Dockergento MySQL Controller

Provides MySQL database management in Hiberus Dockergento environments using `hm mysql` commands for importing dumps, executing queries, and database administration.

## When to Use

- Importing SQL dump files
- Executing SQL queries
- Database restoration
- Data migrations
- Database cleanup operations
- Query debugging
- Performance analysis
- Schema modifications
- Troubleshooting database issues
- Database initialization

## Requirements

- Hiberus Dockergento environment (https://github.com/hiberus-magento/hiberus-dockergento)
- `hm` CLI tool installed and configured
- Docker containers running (mysql container)
- Appropriate database credentials
- SQL dump files (for import operations)

## Execution Workflow

1. **Prepare operation**
   - Identify database operation needed
   - Prepare SQL files or queries
   - Verify database credentials
   - Check disk space (for imports)

2. **Backup current state** (if destructive)
   - Create database backup
   - Verify backup integrity
   - Document restore procedure

3. **Execute operation**
   - Use appropriate `hm mysql` command
   - Monitor execution progress
   - Capture output/errors
   - Verify completion

4. **Verify results**
   - Check operation success
   - Validate data integrity
   - Test application functionality
   - Document changes

5. **Cleanup**
   - Remove temporary files
   - Clear old backups (if retention policy)
   - Update documentation

## Command Reference

### Basic MySQL Access

```bash
# Interactive MySQL shell
hm mysql

# Execute query from command line
hm mysql -e "SELECT * FROM admin_user LIMIT 5"

# Execute SQL file
hm mysql -i dump.sql

# Import compressed dump
gunzip < dump.sql.gz | hm mysql -i
```

### Database Selection

```bash
# Connect to specific database
hm mysql magento

# Execute query on specific database
hm mysql -e "SHOW TABLES" magento

# Import into specific database
hm mysql -i dump.sql magento
```

### Query Options

```bash
# Execute query and format output
hm mysql -e "SELECT entity_id, sku, name FROM catalog_product_entity LIMIT 10" --table

# Execute query with vertical output
hm mysql -e "SHOW CREATE TABLE admin_user\G"

# Execute query without column headers
hm mysql -e "SELECT sku FROM catalog_product_entity" -N

# Batch mode (no formatting)
hm mysql -B -e "SELECT COUNT(*) FROM sales_order"
```

## Implementation Patterns

### Pattern 1: Import SQL Dump

```bash
#!/bin/bash
# Script: import_database.sh

set -e

DUMP_FILE="${1}"

if [ -z "${DUMP_FILE}" ]; then
    echo "Usage: $0 <dump_file.sql[.gz]>"
    exit 1
fi

if [ ! -f "${DUMP_FILE}" ]; then
    echo "ERROR: Dump file not found: ${DUMP_FILE}"
    exit 1
fi

echo "Importing database from: ${DUMP_FILE}"

# Check file size
FILE_SIZE=$(du -h "${DUMP_FILE}" | cut -f1)
echo "File size: ${FILE_SIZE}"

# Backup current database first
echo "Creating backup of current database..."
BACKUP_FILE="backup_before_import_$(date +%Y%m%d_%H%M%S).sql.gz"
hm mysqldump | gzip > "${BACKUP_FILE}"
echo "Backup saved: ${BACKUP_FILE}"

# Import based on file extension
if [[ "${DUMP_FILE}" == *.gz ]]; then
    echo "Importing compressed dump..."
    gunzip < "${DUMP_FILE}" | hm mysql -i
else
    echo "Importing uncompressed dump..."
    hm mysql -i "${DUMP_FILE}"
fi

echo "Import completed successfully"

# Verify import
TABLES_COUNT=$(hm mysql -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='magento'" -N)
echo "Tables imported: ${TABLES_COUNT}"

# Post-import tasks
echo "Running post-import tasks..."
hm bash php bin/magento setup:upgrade
hm bash php bin/magento cache:flush

echo "Database import completed"
```

### Pattern 2: Execute SQL Queries

```bash
#!/bin/bash
# Script: execute_queries.sh

set -e

QUERY_FILE="${1}"

if [ -z "${QUERY_FILE}" ]; then
    echo "Usage: $0 <queries.sql>"
    exit 1
fi

if [ ! -f "${QUERY_FILE}" ]; then
    echo "ERROR: Query file not found: ${QUERY_FILE}"
    exit 1
fi

echo "Executing queries from: ${QUERY_FILE}"

# Show queries to be executed
echo "Queries to execute:"
cat "${QUERY_FILE}"
echo ""

echo "Press Enter to continue or Ctrl+C to cancel..."
read

# Execute queries
if hm mysql -i "${QUERY_FILE}"; then
    echo "✓ Queries executed successfully"
else
    echo "✗ Query execution failed"
    exit 1
fi

echo "Query execution completed"
```

### Pattern 3: Database Cleanup

```bash
#!/bin/bash
# Script: cleanup_database.sh

set -e

echo "Database cleanup operations..."

# Create cleanup SQL
cat > /tmp/cleanup.sql <<'EOF'
-- Truncate log tables
TRUNCATE TABLE report_event;
TRUNCATE TABLE report_viewed_product_index;
TRUNCATE TABLE customer_log;
TRUNCATE TABLE customer_visitor;

-- Clean old sessions (older than 30 days)
DELETE FROM session WHERE updated_at < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Clean old quotes (older than 90 days)
DELETE FROM quote WHERE updated_at < DATE_SUB(NOW(), INTERVAL 90 DAY) AND is_active = 0;

-- Clean old carts
DELETE FROM quote_item WHERE quote_id NOT IN (SELECT entity_id FROM quote);
DELETE FROM quote_address WHERE quote_id NOT IN (SELECT entity_id FROM quote);

-- Optimize tables
OPTIMIZE TABLE report_event;
OPTIMIZE TABLE customer_log;
OPTIMIZE TABLE session;
OPTIMIZE TABLE quote;
EOF

echo "Cleanup queries created"
echo ""

# Show cleanup operations
cat /tmp/cleanup.sql
echo ""

echo "WARNING: This will delete data. Continue? (yes/no)"
read CONFIRM

if [ "${CONFIRM}" != "yes" ]; then
    echo "Cleanup cancelled"
    exit 0
fi

# Backup before cleanup
echo "Creating backup..."
hm mysqldump | gzip > "backup_before_cleanup_$(date +%Y%m%d_%H%M%S).sql.gz"

# Execute cleanup
echo "Executing cleanup..."
hm mysql -i /tmp/cleanup.sql

# Remove temp file
rm /tmp/cleanup.sql

echo "Database cleanup completed"

# Show freed space
echo "Database size after cleanup:"
hm mysql -e "
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'magento'
GROUP BY table_schema;
"
```

### Pattern 4: Database Analysis

```bash
#!/bin/bash
# Script: analyze_database.sh

set -e

echo "=== Database Analysis Report ==="
echo ""

# Database size
echo "1. Database Size:"
hm mysql -e "
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'magento'
GROUP BY table_schema;
" --table
echo ""

# Largest tables
echo "2. Largest Tables (Top 10):"
hm mysql -e "
SELECT 
    table_name AS 'Table',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)',
    table_rows AS 'Rows'
FROM information_schema.tables
WHERE table_schema = 'magento'
ORDER BY (data_length + index_length) DESC
LIMIT 10;
" --table
echo ""

# Tables without primary key
echo "3. Tables Without Primary Key:"
hm mysql -e "
SELECT 
    t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.key_column_usage k 
    ON t.table_schema = k.table_schema 
    AND t.table_name = k.table_name 
    AND k.constraint_name = 'PRIMARY'
WHERE t.table_schema = 'magento' 
    AND k.constraint_name IS NULL
    AND t.table_type = 'BASE TABLE';
" --table
echo ""

# Index statistics
echo "4. Index Usage Statistics:"
hm mysql -e "
SELECT 
    table_name AS 'Table',
    index_name AS 'Index',
    ROUND(index_length / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.statistics
WHERE table_schema = 'magento'
GROUP BY table_name, index_name
ORDER BY index_length DESC
LIMIT 10;
" --table
echo ""

# Database connections
echo "5. Current Connections:"
hm mysql -e "SHOW PROCESSLIST;" --table
echo ""

echo "Analysis completed"
```

### Pattern 5: Data Migration

```bash
#!/bin/bash
# Script: migrate_data.sh

set -e

SOURCE_TABLE="${1}"
TARGET_TABLE="${2}"

if [ -z "${SOURCE_TABLE}" ] || [ -z "${TARGET_TABLE}" ]; then
    echo "Usage: $0 <source_table> <target_table>"
    exit 1
fi

echo "Migrating data from ${SOURCE_TABLE} to ${TARGET_TABLE}..."

# Create migration SQL
cat > /tmp/migration.sql <<EOF
-- Create target table if not exists (copy structure)
CREATE TABLE IF NOT EXISTS ${TARGET_TABLE} LIKE ${SOURCE_TABLE};

-- Migrate data
INSERT INTO ${TARGET_TABLE} SELECT * FROM ${SOURCE_TABLE};

-- Verify migration
SELECT 
    '${SOURCE_TABLE}' AS source_table, 
    COUNT(*) AS source_count 
FROM ${SOURCE_TABLE}
UNION ALL
SELECT 
    '${TARGET_TABLE}' AS target_table, 
    COUNT(*) AS target_count 
FROM ${TARGET_TABLE};
EOF

echo "Migration plan:"
cat /tmp/migration.sql
echo ""

echo "Continue? (yes/no)"
read CONFIRM

if [ "${CONFIRM}" != "yes" ]; then
    echo "Migration cancelled"
    exit 0
fi

# Execute migration
hm mysql -i /tmp/migration.sql

# Cleanup
rm /tmp/migration.sql

echo "Data migration completed"
```

## Common Operations

### Database Information

```bash
# Show databases
hm mysql -e "SHOW DATABASES"

# Show tables
hm mysql -e "SHOW TABLES" magento

# Show table structure
hm mysql -e "DESCRIBE admin_user" magento

# Show table creation statement
hm mysql -e "SHOW CREATE TABLE admin_user\G" magento

# Show table indexes
hm mysql -e "SHOW INDEX FROM sales_order" magento

# Show table status
hm mysql -e "SHOW TABLE STATUS LIKE 'catalog_product_entity'\G" magento
```

### Data Queries

```bash
# Count records
hm mysql -e "SELECT COUNT(*) FROM sales_order" magento

# Simple SELECT
hm mysql -e "SELECT entity_id, increment_id, created_at FROM sales_order LIMIT 10" magento

# Aggregate query
hm mysql -e "
SELECT 
    status, 
    COUNT(*) as count,
    SUM(grand_total) as total
FROM sales_order 
GROUP BY status
" magento --table

# Join query
hm mysql -e "
SELECT 
    o.increment_id,
    o.created_at,
    c.email
FROM sales_order o
LEFT JOIN customer_entity c ON o.customer_id = c.entity_id
LIMIT 10
" magento --table
```

### Data Modification

```bash
# Update records
hm mysql -e "UPDATE core_config_data SET value='0' WHERE path='dev/debug/profiler'"

# Delete records
hm mysql -e "DELETE FROM customer_log WHERE last_login_at < DATE_SUB(NOW(), INTERVAL 1 YEAR)"

# Insert records
hm mysql -e "INSERT INTO admin_user (username, email) VALUES ('test', 'test@example.com')"

# Truncate table
hm mysql -e "TRUNCATE TABLE customer_visitor"
```

### Database Maintenance

```bash
# Optimize tables
hm mysql -e "OPTIMIZE TABLE sales_order, sales_order_item" magento

# Repair table
hm mysql -e "REPAIR TABLE sales_order" magento

# Analyze table
hm mysql -e "ANALYZE TABLE sales_order" magento

# Check table
hm mysql -e "CHECK TABLE sales_order" magento
```

## Performance Optimization

### Analyze Slow Queries

```bash
# Enable slow query log
hm mysql -e "SET GLOBAL slow_query_log = 'ON'"
hm mysql -e "SET GLOBAL long_query_time = 2"

# Show slow query log location
hm mysql -e "SHOW VARIABLES LIKE 'slow_query_log%'"

# Analyze query performance
hm mysql -e "EXPLAIN SELECT * FROM sales_order WHERE status='pending'\G"
```

### Index Management

```bash
# Create index
hm mysql -e "CREATE INDEX idx_sku ON catalog_product_entity(sku)"

# Drop index
hm mysql -e "DROP INDEX idx_sku ON catalog_product_entity"

# Show index usage
hm mysql -e "
SELECT 
    table_name,
    index_name,
    seq_in_index,
    column_name
FROM information_schema.statistics
WHERE table_schema = 'magento'
    AND table_name = 'sales_order'
ORDER BY table_name, index_name, seq_in_index;
" --table
```

### Table Optimization

```bash
# Find tables needing optimization
hm mysql -e "
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
    ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)',
    ROUND(data_free / 1024 / 1024, 2) AS 'Free (MB)'
FROM information_schema.tables
WHERE table_schema = 'magento'
    AND data_free > 0
ORDER BY data_free DESC;
" --table

# Optimize all tables
hm mysql -e "
SELECT CONCAT('OPTIMIZE TABLE ', table_name, ';') AS 'Optimize Statements'
FROM information_schema.tables
WHERE table_schema = 'magento'
" -N | hm mysql -i
```

## Troubleshooting

### Connection Issues

```bash
# Test connection
hm mysql -e "SELECT 1"

# Show connection parameters
hm mysql -e "SHOW VARIABLES LIKE '%host%'"
hm mysql -e "SHOW VARIABLES LIKE '%port%'"

# Show current user
hm mysql -e "SELECT USER(), CURRENT_USER()"

# Show grants
hm mysql -e "SHOW GRANTS FOR CURRENT_USER()"
```

### Import Errors

```bash
# Import with verbose errors
hm mysql -v -v -v -i dump.sql 2>&1 | tee import_errors.log

# Skip errors during import (use cautiously)
hm mysql --force -i dump.sql

# Import in batches (for large files)
split -l 10000 large_dump.sql split_
for file in split_*; do
    echo "Importing ${file}..."
    hm mysql -i "${file}"
done
```

### Lock Issues

```bash
# Show locked tables
hm mysql -e "SHOW OPEN TABLES WHERE In_use > 0"

# Show running queries
hm mysql -e "SHOW FULL PROCESSLIST"

# Kill query
hm mysql -e "KILL QUERY <process_id>"

# Kill connection
hm mysql -e "KILL CONNECTION <process_id>"
```

### Corruption Issues

```bash
# Check all tables
hm mysql -e "
SELECT CONCAT('CHECK TABLE ', table_name, ';')
FROM information_schema.tables
WHERE table_schema = 'magento'
" -N | hm mysql -i

# Repair corrupted tables
hm mysql -e "REPAIR TABLE sales_order"
```

## Best Practices

1. **Always backup before destructive operations**
2. **Test queries on non-production first**
3. **Use transactions for multi-query operations**
4. **Monitor disk space before imports**
5. **Verify import success with counts**
6. **Use compression for large dumps**
7. **Document database changes**
8. **Check slow query log regularly**
9. **Optimize tables periodically**
10. **Use prepared statements for security**

## Common Pitfalls

### ❌ Don't

- Don't import without backup
- Don't ignore import errors
- Don't modify production without testing
- Don't use `--force` without understanding
- Don't forget to optimize after large operations
- Don't hardcode passwords in scripts

### ✅ Do

- Do backup before every import
- Do verify import integrity
- Do test queries in development
- Do handle errors properly
- Do optimize tables after modifications
- Do use environment variables for credentials

## Security Considerations

### Secure Credential Handling

```bash
# Use environment variables
export MYSQL_PWD='password'
hm mysql -e "SELECT 1"

# Avoid credentials in command history
hm mysql -e "SELECT 1" # (credentials from env.php)

# Use .my.cnf file
cat > ~/.my.cnf <<EOF
[client]
user=magento
password=magento
EOF
chmod 600 ~/.my.cnf
```

### SQL Injection Prevention

```bash
# Bad: SQL injection vulnerable
USER_INPUT="admin' OR '1'='1"
hm mysql -e "SELECT * FROM admin_user WHERE username='${USER_INPUT}'"

# Good: Use parameterized queries via script
php -r "
\$pdo = new PDO('mysql:host=mysql;dbname=magento', 'magento', 'magento');
\$stmt = \$pdo->prepare('SELECT * FROM admin_user WHERE username = ?');
\$stmt->execute(['admin']);
print_r(\$stmt->fetchAll());
"
```

## Quality Bar

- Commands execute without errors
- Database operations complete successfully
- Backups created before destructive operations
- Import integrity verified
- Query results validated
- Performance acceptable
- Errors handled appropriately
- Documentation updated
- Security best practices followed
- Team trained on procedures
