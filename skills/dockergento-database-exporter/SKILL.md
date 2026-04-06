---
name: dockergento-database-exporter
description: Extracts and secures database schemas autonomously before dangerous refactorings, ensuring reliable restore points. Uses hm mysqldump to create timestamped backups with compression and verification.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Dockergento Database Exporter

Automatically extracts and secures Magento database schemas using `hm mysqldump` before executing risky operations, providing reliable restore points with verification and compression.

## When to Use

- Before database schema migrations
- Before running destructive SQL operations
- Before module uninstalls
- Before major version upgrades
- Before data cleanup operations
- Before testing risky refactorings
- Daily backup automation
- Pre-deployment snapshots

## Requirements

- Hiberus Dockergento environment (https://github.com/hiberus-magento/hiberus-dockergento)
- `hm` CLI tool installed and configured
- Docker containers running
- Sufficient disk space for dumps

## Execution Workflow

1. **Assess operation risk**
   - Identify if operation is destructive
   - Check if backup exists recently
   - Verify available disk space
   - Determine compression needs

2. **Create backup directory**
   - Use timestamped directory names
   - Organize by date and operation type
   - Keep backup metadata

3. **Execute mysqldump**
   - Use `hm mysqldump` command
   - Add compression if needed
   - Include all tables or specific ones
   - Capture output and errors

4. **Verify backup**
   - Check file size (not empty)
   - Verify SQL syntax (header check)
   - Test restoration (dry-run if critical)
   - Log backup metadata

5. **Provide restore instructions**
   - Document restore command
   - Save backup location
   - Include rollback procedure

## Command Reference

### Basic Database Dump

```bash
# Full database dump
hm mysqldump > backup.sql

# Dump to specific file with timestamp
hm mysqldump > "backup_$(date +%Y%m%d_%H%M%S).sql"

# Compressed dump
hm mysqldump | gzip > backup.sql.gz
```

### Advanced Usage

```bash
# Dump specific tables only
hm mysqldump magento sales_order sales_order_item > orders_backup.sql

# Dump structure only (no data)
hm mysqldump --no-data > schema_only.sql

# Dump data only (no structure)
hm mysqldump --no-create-info > data_only.sql

# Skip certain tables
hm mysqldump --ignore-table=magento.cache --ignore-table=magento.session > backup_no_cache.sql
```

### Database Restoration

```bash
# Restore from dump
hm mysql -i backup.sql

# Restore from compressed dump
gunzip < backup.sql.gz | hm mysql -i

# Restore specific database
hm mysql -i backup.sql magento
```

## Implementation Patterns

### Pattern 1: Pre-Migration Backup

```bash
#!/bin/bash
# Script: pre_migration_backup.sh

set -e

OPERATION="schema_migration"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="backups/${OPERATION}"
BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.sql.gz"

echo "Creating backup before ${OPERATION}..."

# Create backup directory
mkdir -p "${BACKUP_DIR}"

# Execute dump with compression
hm mysqldump | gzip > "${BACKUP_FILE}"

# Verify backup
if [ ! -s "${BACKUP_FILE}" ]; then
    echo "ERROR: Backup file is empty"
    exit 1
fi

# Check if it's valid SQL
if ! gunzip < "${BACKUP_FILE}" | head -n 20 | grep -q "MySQL dump"; then
    echo "ERROR: Backup file doesn't appear to be a valid SQL dump"
    exit 1
fi

# Log backup metadata
cat > "${BACKUP_DIR}/backup_${TIMESTAMP}.meta" <<EOF
Timestamp: ${TIMESTAMP}
Operation: ${OPERATION}
File: ${BACKUP_FILE}
Size: $(du -h "${BACKUP_FILE}" | cut -f1)
Created: $(date)
Restore: gunzip < ${BACKUP_FILE} | hm mysql -i
EOF

echo "Backup created successfully: ${BACKUP_FILE}"
echo "To restore: gunzip < ${BACKUP_FILE} | hm mysql -i"
```

### Pattern 2: Automated Daily Backups

```bash
#!/bin/bash
# Script: daily_backup.sh
# Add to crontab: 0 2 * * * /path/to/daily_backup.sh

set -e

BACKUP_BASE="/var/backups/magento"
DATE=$(date +%Y%m%d)
BACKUP_FILE="${BACKUP_BASE}/${DATE}_daily.sql.gz"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "${BACKUP_BASE}"

# Create backup
echo "Starting daily backup: ${DATE}"
hm mysqldump | gzip > "${BACKUP_FILE}"

# Verify backup size (should be > 1MB)
FILESIZE=$(stat -f%z "${BACKUP_FILE}" 2>/dev/null || stat -c%s "${BACKUP_FILE}")
if [ "${FILESIZE}" -lt 1048576 ]; then
    echo "ERROR: Backup file too small (${FILESIZE} bytes)"
    exit 1
fi

# Clean old backups
find "${BACKUP_BASE}" -name "*_daily.sql.gz" -mtime +${RETENTION_DAYS} -delete

echo "Daily backup completed: ${BACKUP_FILE} ($(du -h "${BACKUP_FILE}" | cut -f1))"

# Optional: Upload to remote storage
# aws s3 cp "${BACKUP_FILE}" "s3://backups/magento/${DATE}_daily.sql.gz"
```

### Pattern 3: Pre-Deployment Snapshot

```bash
#!/bin/bash
# Script: pre_deploy_snapshot.sh

set -e

DEPLOY_ID="${1:-$(date +%Y%m%d_%H%M%S)}"
BACKUP_DIR="deployments/${DEPLOY_ID}"
BACKUP_FILE="${BACKUP_DIR}/pre_deploy.sql.gz"

echo "Creating pre-deployment snapshot: ${DEPLOY_ID}"

# Create deployment backup directory
mkdir -p "${BACKUP_DIR}"

# Capture current git commit
git rev-parse HEAD > "${BACKUP_DIR}/git_commit.txt"

# Capture current module list
php bin/magento module:status > "${BACKUP_DIR}/module_status.txt"

# Database backup
echo "Dumping database..."
hm mysqldump | gzip > "${BACKUP_FILE}"

# Verify
if [ ! -s "${BACKUP_FILE}" ]; then
    echo "ERROR: Database backup failed"
    exit 1
fi

# Create restore script
cat > "${BACKUP_DIR}/restore.sh" <<'EOF'
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
BACKUP_FILE="${SCRIPT_DIR}/pre_deploy.sql.gz"

echo "Restoring database from: ${BACKUP_FILE}"
gunzip < "${BACKUP_FILE}" | hm mysql -i

echo "Database restored successfully"
echo "Remember to:"
echo "  1. Run: bin/magento setup:upgrade"
echo "  2. Clear cache: bin/magento cache:flush"
echo "  3. Reindex: bin/magento indexer:reindex"
EOF

chmod +x "${BACKUP_DIR}/restore.sh"

# Save metadata
cat > "${BACKUP_DIR}/metadata.json" <<EOF
{
  "deploy_id": "${DEPLOY_ID}",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "git_commit": "$(cat ${BACKUP_DIR}/git_commit.txt)",
  "backup_file": "${BACKUP_FILE}",
  "backup_size": "$(du -h "${BACKUP_FILE}" | cut -f1)",
  "restore_command": "cd ${BACKUP_DIR} && ./restore.sh"
}
EOF

echo "Pre-deployment snapshot completed: ${BACKUP_DIR}"
echo "To restore: cd ${BACKUP_DIR} && ./restore.sh"
```

### Pattern 4: Selective Table Backup

```bash
#!/bin/bash
# Script: backup_critical_tables.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="backups/critical_tables"
BACKUP_FILE="${BACKUP_DIR}/critical_${TIMESTAMP}.sql.gz"

# Define critical tables
CRITICAL_TABLES=(
    "sales_order"
    "sales_order_item"
    "sales_order_address"
    "sales_order_payment"
    "quote"
    "quote_item"
    "customer_entity"
    "customer_address_entity"
)

echo "Backing up critical tables: ${#CRITICAL_TABLES[@]} tables"

# Create backup directory
mkdir -p "${BACKUP_DIR}"

# Dump critical tables
hm mysqldump magento "${CRITICAL_TABLES[@]}" | gzip > "${BACKUP_FILE}"

# Verify
if [ ! -s "${BACKUP_FILE}" ]; then
    echo "ERROR: Backup failed"
    exit 1
fi

# Save table list
printf '%s\n' "${CRITICAL_TABLES[@]}" > "${BACKUP_DIR}/tables_${TIMESTAMP}.txt"

echo "Critical tables backed up: ${BACKUP_FILE}"
echo "Size: $(du -h "${BACKUP_FILE}" | cut -f1)"
```

### Pattern 5: Verification and Testing

```php
<?php
// File: scripts/verify_backup.php

declare(strict_types=1);

class BackupVerifier
{
    private const MIN_BACKUP_SIZE = 1048576; // 1MB
    
    public function verifyBackup(string $backupFile): array
    {
        $errors = [];
        $warnings = [];
        
        // Check file exists
        if (!file_exists($backupFile)) {
            $errors[] = "Backup file not found: {$backupFile}";
            return ['errors' => $errors, 'warnings' => $warnings, 'valid' => false];
        }
        
        // Check file size
        $fileSize = filesize($backupFile);
        if ($fileSize < self::MIN_BACKUP_SIZE) {
            $errors[] = "Backup file too small: " . $this->formatBytes($fileSize);
        }
        
        // Check if gzipped
        $isGzipped = str_ends_with($backupFile, '.gz');
        
        // Read first few lines
        $handle = $isGzipped ? gzopen($backupFile, 'rb') : fopen($backupFile, 'rb');
        if (!$handle) {
            $errors[] = "Cannot open backup file";
            return ['errors' => $errors, 'warnings' => $warnings, 'valid' => false];
        }
        
        $header = '';
        for ($i = 0; $i < 20; $i++) {
            $line = $isGzipped ? gzgets($handle) : fgets($handle);
            if ($line === false) break;
            $header .= $line;
        }
        
        if ($isGzipped) {
            gzclose($handle);
        } else {
            fclose($handle);
        }
        
        // Verify SQL dump header
        if (!str_contains($header, 'MySQL dump')) {
            $errors[] = "File doesn't appear to be a MySQL dump";
        }
        
        // Check for common tables
        $commonTables = ['admin_user', 'store', 'store_website', 'customer_entity'];
        $foundTables = 0;
        foreach ($commonTables as $table) {
            if (str_contains($header, "CREATE TABLE `{$table}`")) {
                $foundTables++;
            }
        }
        
        if ($foundTables === 0) {
            $warnings[] = "No common Magento tables found in dump header";
        }
        
        $valid = empty($errors);
        
        return [
            'valid' => $valid,
            'errors' => $errors,
            'warnings' => $warnings,
            'file_size' => $fileSize,
            'file_size_formatted' => $this->formatBytes($fileSize),
            'is_compressed' => $isGzipped,
        ];
    }
    
    private function formatBytes(int $bytes): string
    {
        $units = ['B', 'KB', 'MB', 'GB'];
        $power = $bytes > 0 ? floor(log($bytes, 1024)) : 0;
        return number_format($bytes / pow(1024, $power), 2) . ' ' . $units[$power];
    }
}

// Usage
$verifier = new BackupVerifier();
$result = $verifier->verifyBackup($argv[1] ?? 'backup.sql.gz');

if ($result['valid']) {
    echo "✓ Backup is valid\n";
    echo "  Size: {$result['file_size_formatted']}\n";
    echo "  Compressed: " . ($result['is_compressed'] ? 'Yes' : 'No') . "\n";
} else {
    echo "✗ Backup verification failed\n";
    foreach ($result['errors'] as $error) {
        echo "  ERROR: {$error}\n";
    }
}

if (!empty($result['warnings'])) {
    foreach ($result['warnings'] as $warning) {
        echo "  WARNING: {$warning}\n";
    }
}

exit($result['valid'] ? 0 : 1);
```

## Backup Strategy

### Retention Policy

```bash
# Keep backups for different durations
BACKUP_BASE="/var/backups/magento"

# Hourly: Keep 24 hours
find "${BACKUP_BASE}/hourly" -name "*.sql.gz" -mtime +1 -delete

# Daily: Keep 30 days
find "${BACKUP_BASE}/daily" -name "*.sql.gz" -mtime +30 -delete

# Weekly: Keep 12 weeks
find "${BACKUP_BASE}/weekly" -name "*.sql.gz" -mtime +84 -delete

# Monthly: Keep 12 months
find "${BACKUP_BASE}/monthly" -name "*.sql.gz" -mtime +365 -delete

# Pre-deployment: Keep forever
# (manually clean when deployment is obsolete)
```

### Backup Size Optimization

```bash
# Exclude cache and session tables
hm mysqldump \
    --ignore-table=magento.cache \
    --ignore-table=magento.cache_tag \
    --ignore-table=magento.session \
    --ignore-table=magento.customer_visitor \
    --ignore-table=magento.report_event \
    | gzip -9 > optimized_backup.sql.gz

# Structure only (for version control)
hm mysqldump --no-data > schema.sql

# Data only for specific tables
hm mysqldump --no-create-info magento sales_order > orders_data.sql
```

## Restore Procedures

### Full Database Restore

```bash
#!/bin/bash
# Script: restore_full.sh

set -e

BACKUP_FILE="${1}"

if [ -z "${BACKUP_FILE}" ]; then
    echo "Usage: $0 <backup_file.sql.gz>"
    exit 1
fi

if [ ! -f "${BACKUP_FILE}" ]; then
    echo "ERROR: Backup file not found: ${BACKUP_FILE}"
    exit 1
fi

echo "WARNING: This will replace the entire database!"
echo "Backup file: ${BACKUP_FILE}"
echo "Press Ctrl+C to cancel, or wait 10 seconds to proceed..."
sleep 10

echo "Restoring database..."
gunzip < "${BACKUP_FILE}" | hm mysql -i

echo "Database restored successfully"
echo "Running post-restore tasks..."

# Post-restore cleanup
hm bash php bin/magento setup:upgrade
hm bash php bin/magento cache:flush
hm bash php bin/magento indexer:reindex

echo "Restore completed successfully"
```

### Selective Table Restore

```bash
#!/bin/bash
# Script: restore_tables.sh

BACKUP_FILE="${1}"
shift
TABLES=("$@")

if [ ${#TABLES[@]} -eq 0 ]; then
    echo "Usage: $0 <backup_file.sql.gz> table1 table2 ..."
    exit 1
fi

echo "Extracting tables: ${TABLES[*]}"

# Extract and restore specific tables
gunzip < "${BACKUP_FILE}" | \
    sed -n "/CREATE TABLE \`${TABLES[0]}\`/,/UNLOCK TABLES/p" | \
    hm mysql -i

echo "Tables restored: ${TABLES[*]}"
```

## Best Practices

1. **Always backup before destructive operations**
2. **Verify backup integrity immediately** after creation
3. **Compress large backups** to save disk space
4. **Use timestamped filenames** for easy identification
5. **Store backups outside Docker volumes** (persistent storage)
6. **Test restore procedures regularly** (quarterly)
7. **Document restore commands** with each backup
8. **Implement retention policies** to manage disk usage
9. **Monitor backup failures** (alert on errors)
10. **Exclude transient data** (cache, sessions) from production backups

## Common Pitfalls

### ❌ Don't

- Don't backup to Docker volumes (data loss on rebuild)
- Don't skip verification after backup
- Don't store backups only locally (hardware failure)
- Don't backup without compression (disk space)
- Don't forget to test restore procedures
- Don't mix structure and data backups

### ✅ Do

- Store backups on persistent host filesystem
- Verify every backup immediately
- Upload critical backups to remote storage
- Use compression for large databases
- Test restoration quarterly
- Separate structure (schema.sql) from data

## Automation Examples

### Pre-Commit Hook

```bash
# File: .git/hooks/pre-commit
#!/bin/bash

# Backup before commits that touch migrations
if git diff --cached --name-only | grep -q "Setup/.*\.php\|db_schema\.xml"; then
    echo "Detected database migration changes, creating backup..."
    ./scripts/pre_migration_backup.sh
fi
```

### Deployment Integration

```yaml
# File: .gitlab-ci.yml
before_script:
  - ./scripts/pre_deploy_snapshot.sh "${CI_COMMIT_SHORT_SHA}"

deploy:
  script:
    - bin/magento setup:upgrade
    - bin/magento cache:flush
  after_script:
    - echo "Backup available at: deployments/${CI_COMMIT_SHORT_SHA}/restore.sh"
```

## Quality Bar

- Backup created successfully without errors
- Backup file size > 1MB (non-empty)
- Valid SQL dump header present
- Compression applied for large databases
- Verification script confirms integrity
- Restore command documented
- Backup metadata saved (timestamp, size, operation)
- Retention policy configured
- Post-restore procedures documented
