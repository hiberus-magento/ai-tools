---
name: dockergento-shell-executor
description: Executes shell commands inside Dockergento containers using hm bash and hm exec. Use when running PHP scripts, CLI commands, composer operations, or debugging container environments.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Dockergento Shell Executor

Provides shell access to Dockergento containers via `hm bash` and `hm exec` commands for executing PHP scripts, Magento CLI commands, composer operations, and container debugging.

## When to Use

- Running Magento CLI commands
- Executing PHP scripts
- Composer dependency management
- Container environment inspection
- File system operations
- Debugging container state
- Running custom scripts
- Database operations (via CLI)
- Cache operations
- Module management

## Requirements

- Hiberus Dockergento environment (https://github.com/hiberus-magento/hiberus-dockergento)
- `hm` CLI tool installed and configured
- Docker containers running
- Proper container permissions

## Execution Workflow

1. **Identify operation**
   - Determine target container (default: php)
   - Plan command execution
   - Check for dependencies
   - Consider working directory

2. **Choose execution method**
   - `hm bash`: Interactive shell or single command
   - `hm exec`: Direct command execution
   - Consider output handling

3. **Execute command**
   - Run via appropriate method
   - Capture output if needed
   - Handle errors properly
   - Log execution results

4. **Verify results**
   - Check command exit code
   - Validate output
   - Verify side effects
   - Document outcomes

## Command Reference

### Basic Shell Access

```bash
# Interactive bash shell in PHP container
hm bash

# Run single command in PHP container
hm bash <command>

# Examples
hm bash php -v
hm bash ls -la
hm bash pwd
```

### Execute Commands

```bash
# Direct command execution
hm exec <command>

# Execute in specific container
hm bash -c <container> <command>

# Examples
hm bash php bin/magento cache:flush
hm bash composer install
hm bash -c nginx nginx -v
```

### Container Selection

```bash
# PHP container (default)
hm bash php -v

# Nginx container
hm bash -c nginx nginx -t

# Varnish container
hm bash -c varnish varnishd -V

# MySQL container
hm bash -c mysql mysql --version

# Redis container
hm bash -c redis redis-cli --version

# Elasticsearch container
hm bash -c elasticsearch curl localhost:9200
```

## Implementation Patterns

### Pattern 1: Magento CLI Commands

```bash
#!/bin/bash
# Script: magento_operations.sh

set -e

echo "Running Magento operations..."

# Cache operations
echo "Flushing cache..."
hm bash php bin/magento cache:flush

# Module operations
echo "Enabling custom modules..."
hm bash php bin/magento module:enable Vendor_Module
hm bash php bin/magento setup:upgrade

# Compilation
echo "Running compilation..."
hm bash php bin/magento setup:di:compile

# Static content deployment
echo "Deploying static content..."
hm bash php bin/magento setup:static-content:deploy -f

# Reindexing
echo "Running reindex..."
hm bash php bin/magento indexer:reindex

echo "Magento operations completed"
```

### Pattern 2: Composer Management

```bash
#!/bin/bash
# Script: composer_operations.sh

set -e

OPERATION="${1:-install}"

echo "Running composer ${OPERATION}..."

case "${OPERATION}" in
    install)
        hm bash composer install --no-interaction
        ;;
    
    update)
        hm bash composer update --no-interaction
        ;;
    
    require)
        PACKAGE="${2}"
        if [ -z "${PACKAGE}" ]; then
            echo "Usage: $0 require <package>"
            exit 1
        fi
        hm bash composer require "${PACKAGE}"
        ;;
    
    remove)
        PACKAGE="${2}"
        if [ -z "${PACKAGE}" ]; then
            echo "Usage: $0 remove <package>"
            exit 1
        fi
        hm bash composer remove "${PACKAGE}"
        ;;
    
    validate)
        hm bash composer validate
        ;;
    
    diagnose)
        hm bash composer diagnose
        ;;
    
    *)
        echo "Unknown operation: ${OPERATION}"
        exit 1
        ;;
esac

echo "Composer ${OPERATION} completed"
```

### Pattern 3: PHP Script Execution

```bash
#!/bin/bash
# Script: run_php_script.sh

set -e

SCRIPT="${1}"

if [ -z "${SCRIPT}" ]; then
    echo "Usage: $0 <script.php>"
    exit 1
fi

if [ ! -f "${SCRIPT}" ]; then
    echo "ERROR: Script not found: ${SCRIPT}"
    exit 1
fi

echo "Executing PHP script: ${SCRIPT}"

# Copy script to container (if needed)
# docker cp "${SCRIPT}" dockergento_php:/tmp/script.php

# Execute script
hm bash php "${SCRIPT}"

echo "Script execution completed"
```

### Pattern 4: Environment Inspection

```bash
#!/bin/bash
# Script: inspect_environment.sh

set -e

echo "=== Dockergento Environment Inspection ==="
echo ""

# PHP version and modules
echo "PHP Version:"
hm bash php -v
echo ""

echo "PHP Modules:"
hm bash php -m | sort
echo ""

# Composer version
echo "Composer Version:"
hm bash composer --version
echo ""

# Magento version
echo "Magento Version:"
hm bash php bin/magento --version
echo ""

# Database connection
echo "Database Connection:"
hm bash php bin/magento setup:db:status
echo ""

# Redis connection
echo "Redis Connection:"
hm bash -c redis redis-cli ping
echo ""

# Elasticsearch status
echo "Elasticsearch Status:"
hm bash -c elasticsearch curl -s localhost:9200/_cluster/health?pretty
echo ""

# Disk usage
echo "Disk Usage:"
hm bash df -h /var/www/html
echo ""

# Memory usage
echo "Memory Usage:"
hm bash free -h
echo ""

echo "Environment inspection completed"
```

### Pattern 5: Batch Operations

```bash
#!/bin/bash
# Script: batch_operations.sh

set -e

echo "Running batch operations..."

# Array of commands
COMMANDS=(
    "php bin/magento cache:flush"
    "php bin/magento setup:upgrade"
    "php bin/magento setup:di:compile"
    "php bin/magento indexer:reindex"
    "php bin/magento cron:run"
)

# Execute each command
for cmd in "${COMMANDS[@]}"; do
    echo "Executing: ${cmd}"
    if hm bash ${cmd}; then
        echo "✓ Success: ${cmd}"
    else
        echo "✗ Failed: ${cmd}"
        exit 1
    fi
    echo ""
done

echo "All batch operations completed successfully"
```

## Advanced Usage

### Working Directory Control

```bash
# Execute in specific directory
hm bash "cd /var/www/html/vendor && ls -la"

# Multiple commands with directory context
hm bash "cd /tmp && php test.php && rm test.php"
```

### Environment Variables

```bash
# Set environment variables
hm bash "MAGE_MODE=developer php bin/magento cache:flush"

# Multiple variables
hm bash "XDEBUG_MODE=debug php -d memory_limit=4G bin/magento setup:upgrade"
```

### Output Redirection

```bash
# Save output to file (host)
hm bash php bin/magento module:status > module_status.txt

# Save output to file (container)
hm bash "php bin/magento module:status > /tmp/status.txt"

# Copy from container to host
docker cp dockergento_php:/tmp/status.txt ./
```

### Error Handling

```bash
#!/bin/bash
# Script: safe_execution.sh

set -e

# Execute with error handling
if hm bash php bin/magento setup:upgrade 2>&1 | tee upgrade.log; then
    echo "✓ Upgrade successful"
else
    echo "✗ Upgrade failed - check upgrade.log"
    exit 1
fi

# Check exit code
hm bash php bin/magento setup:db:status
EXIT_CODE=$?

if [ ${EXIT_CODE} -eq 0 ]; then
    echo "Database is up to date"
else
    echo "Database needs upgrade"
fi
```

## Common Operations

### Cache Management

```bash
# Flush all caches
hm bash php bin/magento cache:flush

# Flush specific cache types
hm bash php bin/magento cache:flush config layout

# Enable cache types
hm bash php bin/magento cache:enable

# Disable cache types
hm bash php bin/magento cache:disable

# Clean cache
hm bash php bin/magento cache:clean
```

### Module Management

```bash
# List modules
hm bash php bin/magento module:status

# Enable module
hm bash php bin/magento module:enable Vendor_Module

# Disable module
hm bash php bin/magento module:disable Vendor_Module

# Uninstall module
hm bash php bin/magento module:uninstall Vendor_Module
```

### Indexer Management

```bash
# Show indexer status
hm bash php bin/magento indexer:status

# Reindex all
hm bash php bin/magento indexer:reindex

# Reindex specific
hm bash php bin/magento indexer:reindex catalog_product_price

# Set indexer mode
hm bash php bin/magento indexer:set-mode schedule
```

### Configuration Management

```bash
# Show config value
hm bash php bin/magento config:show web/secure/base_url

# Set config value
hm bash php bin/magento config:set web/secure/base_url "https://example.com/"

# Delete config value
hm bash php bin/magento config:delete web/secure/base_url
```

### Cron Management

```bash
# Run cron
hm bash php bin/magento cron:run

# Install cron
hm bash php bin/magento cron:install

# Remove cron
hm bash php bin/magento cron:remove
```

## File Operations

### File Permissions

```bash
# Fix permissions
hm bash chmod -R 775 var generated pub/static pub/media

# Fix ownership
hm bash chown -R www-data:www-data var generated pub/static pub/media

# Check permissions
hm bash ls -la var/ generated/ pub/
```

### File Editing

```bash
# Edit file in container
hm bash vi app/etc/env.php

# Use nano editor
hm bash nano app/etc/env.php

# View file content
hm bash cat app/etc/env.php

# Search in files
hm bash grep -r "search_term" app/code/
```

### File Transfer

```bash
# Copy file from host to container
docker cp local_file.php dockergento_php:/var/www/html/

# Copy file from container to host
docker cp dockergento_php:/var/www/html/var/log/system.log ./

# Copy directory
docker cp ./app/code/Vendor dockergento_php:/var/www/html/app/code/
```

## Debugging Commands

### Check PHP Configuration

```bash
# Show PHP info
hm bash php -i | grep "Configuration File"

# Check specific setting
hm bash php -i | grep memory_limit

# List loaded extensions
hm bash php -m

# Check specific extension
hm bash php -m | grep xdebug
```

### Check Database Connection

```bash
# Test database connection
hm bash php -r "new PDO('mysql:host=mysql;dbname=magento', 'magento', 'magento');"

# Via Magento CLI
hm bash php bin/magento setup:db:status
```

### Check Redis Connection

```bash
# Ping Redis
hm bash -c redis redis-cli ping

# Check Redis info
hm bash -c redis redis-cli info

# Monitor Redis commands
hm bash -c redis redis-cli monitor
```

### Check Logs

```bash
# View system log
hm bash tail -f var/log/system.log

# View exception log
hm bash tail -f var/log/exception.log

# View debug log
hm bash tail -f var/log/debug.log

# View all logs
hm bash tail -f var/log/*.log
```

## Performance Optimization

### Profile PHP Execution

```bash
# Execute with timing
time hm bash php bin/magento indexer:reindex

# Execute with memory profiling
hm bash php -d memory_limit=-1 bin/magento setup:upgrade
```

### Parallel Execution

```bash
#!/bin/bash
# Execute multiple commands in parallel

hm bash php bin/magento indexer:reindex catalog_category_product &
hm bash php bin/magento indexer:reindex catalog_product_category &
hm bash php bin/magento indexer:reindex catalog_product_price &

# Wait for all to complete
wait

echo "All indexers completed"
```

## Testing Integration

### Run Unit Tests

```bash
# Run PHPUnit tests
hm bash vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist

# Run specific test
hm bash vendor/bin/phpunit app/code/Vendor/Module/Test/Unit/
```

### Run Integration Tests

```bash
# Run integration tests
hm bash vendor/bin/phpunit -c dev/tests/integration/phpunit.xml.dist

# Run specific integration test
hm bash vendor/bin/phpunit dev/tests/integration/testsuite/Magento/Catalog/
```

### Code Quality

```bash
# Run PHP Code Sniffer
hm bash vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module/

# Run PHP Mess Detector
hm bash vendor/bin/phpmd app/code/Vendor/Module/ text cleancode,codesize,design

# Run PHPStan
hm bash vendor/bin/phpstan analyse app/code/Vendor/Module/
```

## Best Practices

1. **Use `hm bash` for single commands** (simpler syntax)
2. **Use interactive mode** for exploration (`hm bash` without args)
3. **Always check exit codes** for error handling
4. **Capture output** when needed for logging
5. **Set working directory** explicitly when needed
6. **Use proper error handling** in scripts
7. **Document complex operations** with comments
8. **Test commands** in development first
9. **Use absolute paths** when possible
10. **Clean up temporary files** after operations

## Common Pitfalls

### ❌ Don't

- Don't run destructive commands without backup
- Don't ignore exit codes in scripts
- Don't hard-code container names (use `hm`)
- Don't assume working directory
- Don't forget to handle errors
- Don't run long operations without logging

### ✅ Do

- Do backup before destructive operations
- Do check exit codes and handle errors
- Do use `hm` commands for portability
- Do specify working directory explicitly
- Do implement proper error handling
- Do log important operations

## Troubleshooting

### Command Not Found

```bash
# Check if command exists in container
hm bash which php
hm bash which composer

# Check PATH
hm bash echo $PATH

# Use absolute path
hm bash /usr/local/bin/php -v
```

### Permission Denied

```bash
# Check current user
hm bash whoami

# Check file permissions
hm bash ls -la <file>

# Fix permissions
hm bash chmod +x <file>

# Run as root (if needed)
docker exec -u root dockergento_php <command>
```

### Working Directory Issues

```bash
# Check current directory
hm bash pwd

# Change directory first
hm bash "cd /var/www/html && <command>"

# Use absolute paths
hm bash php /var/www/html/bin/magento cache:flush
```

## Quality Bar

- Commands execute without errors
- Exit codes checked and handled
- Output captured when needed
- Errors logged appropriately
- Working directory managed correctly
- Permissions respected
- Container state verified
- Operations documented
- Scripts tested thoroughly
- Team trained on usage
