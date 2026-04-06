---
name: dockergento-xdebug-toggle
description: Enables or disables Xdebug in Dockergento environment using hm debug-on and hm debug-off commands. Use when debugging PHP code, profiling performance, or optimizing development workflow.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Dockergento Xdebug Toggle

Manages Xdebug activation in Hiberus Dockergento environments using `hm debug-on` and `hm debug-off` commands for efficient PHP debugging and performance profiling.

## When to Use

- Debugging PHP code with breakpoints
- Profiling application performance
- Analyzing code coverage for tests
- Tracing function calls and stack traces
- Investigating complex bugs
- Before/after performance testing
- Development workflow optimization
- Remote debugging sessions

## Requirements

- Hiberus Dockergento environment (https://github.com/hiberus-magento/hiberus-dockergento)
- `hm` CLI tool installed and configured
- Docker containers running
- IDE configured for Xdebug (VS Code, PhpStorm)
- Port 9003 available (Xdebug 3) or 9000 (Xdebug 2)

## Execution Workflow

1. **Assess debugging need**
   - Identify if Xdebug is required
   - Check if already enabled
   - Verify IDE configuration
   - Plan debugging session

2. **Enable Xdebug**
   - Run `hm debug-on`
   - Wait for container restart
   - Verify Xdebug is active
   - Start IDE debugging listener

3. **Debug session**
   - Set breakpoints in IDE
   - Trigger PHP execution
   - Step through code
   - Inspect variables

4. **Disable Xdebug**
   - Run `hm debug-off`
   - Restore normal performance
   - Verify Xdebug is disabled

5. **Optimize workflow**
   - Keep Xdebug off by default
   - Enable only when needed
   - Monitor performance impact

## Command Reference

### Basic Commands

```bash
# Enable Xdebug
hm debug-on

# Disable Xdebug
hm debug-off

# Check Xdebug status
hm bash php -v | grep Xdebug

# Verify Xdebug module loaded
hm bash php -m | grep xdebug

# Show Xdebug configuration
hm bash php --ri xdebug
```

### Verify Xdebug Status

```bash
# Check if Xdebug is enabled
if hm bash php -v | grep -q "Xdebug"; then
    echo "Xdebug is ENABLED"
else
    echo "Xdebug is DISABLED"
fi

# Show Xdebug version
hm bash php -v | grep Xdebug | head -n 1

# List Xdebug settings
hm bash php -i | grep xdebug
```

## Implementation Patterns

### Pattern 1: Automated Debug Session

```bash
#!/bin/bash
# Script: debug_session.sh

set -e

echo "Starting debug session..."

# Enable Xdebug
echo "Enabling Xdebug..."
hm debug-on

# Wait for container to restart
sleep 5

# Verify Xdebug is active
if ! hm bash php -v | grep -q "Xdebug"; then
    echo "ERROR: Xdebug failed to enable"
    exit 1
fi

echo "Xdebug enabled successfully"
echo "Xdebug version: $(hm bash php -v | grep Xdebug | head -n 1)"
echo ""
echo "IDE Setup Instructions:"
echo "  1. Start your IDE's debug listener (Listen for Xdebug)"
echo "  2. Set breakpoints in your code"
echo "  3. Access your site with ?XDEBUG_SESSION_START=PHPSTORM"
echo "  4. When done, run: hm debug-off"
echo ""
echo "Press Enter when done debugging..."
read

# Disable Xdebug
echo "Disabling Xdebug..."
hm debug-off

echo "Debug session completed"
```

### Pattern 2: Performance Testing (Xdebug Off)

```bash
#!/bin/bash
# Script: performance_test.sh

set -e

# Ensure Xdebug is off for accurate performance testing
echo "Disabling Xdebug for performance test..."
hm debug-off

sleep 3

# Verify Xdebug is disabled
if hm bash php -v | grep -q "Xdebug"; then
    echo "ERROR: Xdebug still enabled, aborting test"
    exit 1
fi

echo "Running performance test (Xdebug disabled)..."

# Run performance test
time hm bash php bin/magento cache:flush
time hm bash php bin/magento indexer:reindex

echo "Performance test completed"
```

### Pattern 3: Conditional Debugging

```bash
#!/bin/bash
# Script: conditional_debug.sh

# Enable debugging based on environment variable
if [ "${ENABLE_DEBUG}" = "true" ]; then
    echo "Debug mode enabled"
    hm debug-on
    
    # Set Xdebug session cookie for browser
    export XDEBUG_SESSION=PHPSTORM
else
    echo "Debug mode disabled"
    hm debug-off
fi

# Run command
hm bash php bin/magento "$@"
```

### Pattern 4: Profile Performance

```bash
#!/bin/bash
# Script: profile_script.sh

set -e

PROFILE_DIR="var/profiling"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "Starting profiling session..."

# Enable Xdebug with profiling
hm debug-on

# Create profiling directory
hm bash mkdir -p "${PROFILE_DIR}"

# Set profiling trigger via environment
export XDEBUG_TRIGGER=1

# Run the script to profile
echo "Executing script with profiling enabled..."
hm bash php -d xdebug.mode=profile \
            -d xdebug.output_dir="${PROFILE_DIR}" \
            -d xdebug.profiler_output_name="profile_${TIMESTAMP}.%p.cachegrind" \
            bin/magento indexer:reindex

echo "Profiling data saved to: ${PROFILE_DIR}"
echo "Analyze with: qcachegrind ${PROFILE_DIR}/profile_${TIMESTAMP}.*.cachegrind"

# Disable Xdebug
hm debug-off

echo "Profiling completed"
```

### Pattern 5: IDE Configuration Helper

```bash
#!/bin/bash
# Script: setup_ide_debug.sh

set -e

IDE="${1:-vscode}"

echo "Setting up ${IDE} for Xdebug debugging..."

case "${IDE}" in
    vscode)
        cat > .vscode/launch.json <<'EOF'
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/html": "${workspaceFolder}"
            },
            "log": true,
            "xdebugSettings": {
                "max_data": 65535,
                "show_hidden": 1,
                "max_children": 100,
                "max_depth": 5
            }
        },
        {
            "name": "Launch currently open script",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "port": 9003
        }
    ]
}
EOF
        echo "VS Code configuration created: .vscode/launch.json"
        ;;
        
    phpstorm)
        echo "PhpStorm configuration:"
        echo "  1. Go to: Settings > PHP > Debug"
        echo "  2. Set Xdebug port: 9003"
        echo "  3. Enable 'Can accept external connections'"
        echo "  4. Go to: Settings > PHP > Servers"
        echo "  5. Add server:"
        echo "     - Name: docker"
        echo "     - Host: localhost"
        echo "     - Port: 80"
        echo "     - Use path mappings:"
        echo "       ${PWD} -> /var/www/html"
        echo "  6. Enable toolbar debug listener"
        ;;
        
    *)
        echo "Unknown IDE: ${IDE}"
        echo "Supported: vscode, phpstorm"
        exit 1
        ;;
esac

echo ""
echo "Next steps:"
echo "  1. Run: hm debug-on"
echo "  2. Start IDE debug listener"
echo "  3. Set breakpoints"
echo "  4. Access: http://localhost/?XDEBUG_SESSION_START=PHPSTORM"
```

## Xdebug Configuration

### Check Current Configuration

```bash
# Show all Xdebug settings
hm bash php -i | grep xdebug

# Key settings to verify
hm bash php -i | grep -E "xdebug\.(mode|client_host|client_port|start_with_request)"
```

### Common Xdebug Settings (Xdebug 3)

```ini
; Xdebug 3 configuration (usually in docker-compose.yml)
xdebug.mode=debug,develop,coverage,profile,trace
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
xdebug.start_with_request=trigger
xdebug.idekey=PHPSTORM
xdebug.log=/var/log/xdebug.log
```

## Debugging Workflows

### Workflow 1: Breakpoint Debugging

```bash
# 1. Enable Xdebug
hm debug-on

# 2. Start IDE debug listener
# (Click "Start Listening for PHP Debug Connections" in IDE)

# 3. Access with debug trigger
curl "http://localhost/?XDEBUG_SESSION_START=PHPSTORM"

# 4. Set breakpoints in IDE
# 5. Step through code (F10, F11)
# 6. Inspect variables

# 7. When done, disable Xdebug
hm debug-off
```

### Workflow 2: CLI Script Debugging

```bash
# Enable Xdebug
hm debug-on

# Start IDE listener

# Run CLI script with debug trigger
hm bash php -d xdebug.start_with_request=yes bin/magento catalog:product:import

# Debug in IDE

# Disable Xdebug
hm debug-off
```

### Workflow 3: Remote Debugging

```bash
# Enable Xdebug with remote host
hm debug-on

# Access from remote with SSH tunnel
ssh -R 9003:localhost:9003 user@remote-server

# On remote server, trigger debugging
curl "http://localhost/?XDEBUG_SESSION_START=PHPSTORM"

# Debug locally in IDE
```

## Performance Impact

### Xdebug Overhead

```bash
#!/bin/bash
# Script: measure_xdebug_overhead.sh

set -e

ITERATIONS=10

echo "Measuring Xdebug performance impact..."
echo ""

# Disable Xdebug
hm debug-off
sleep 3

echo "Testing WITHOUT Xdebug..."
TOTAL_WITHOUT=0
for i in $(seq 1 ${ITERATIONS}); do
    START=$(date +%s%N)
    hm bash php bin/magento cache:flush > /dev/null 2>&1
    END=$(date +%s%N)
    DURATION=$(( (END - START) / 1000000 ))
    TOTAL_WITHOUT=$((TOTAL_WITHOUT + DURATION))
    echo "  Iteration ${i}: ${DURATION}ms"
done
AVG_WITHOUT=$((TOTAL_WITHOUT / ITERATIONS))

echo ""
echo "Testing WITH Xdebug..."
hm debug-on
sleep 3

TOTAL_WITH=0
for i in $(seq 1 ${ITERATIONS}); do
    START=$(date +%s%N)
    hm bash php bin/magento cache:flush > /dev/null 2>&1
    END=$(date +%s%N)
    DURATION=$(( (END - START) / 1000000 ))
    TOTAL_WITH=$((TOTAL_WITH + DURATION))
    echo "  Iteration ${i}: ${DURATION}ms"
done
AVG_WITH=$((TOTAL_WITH / ITERATIONS))

# Disable Xdebug
hm debug-off

echo ""
echo "Results:"
echo "  Average WITHOUT Xdebug: ${AVG_WITHOUT}ms"
echo "  Average WITH Xdebug:    ${AVG_WITH}ms"
echo "  Overhead:               $((AVG_WITH - AVG_WITHOUT))ms"
echo "  Slowdown factor:        $((AVG_WITH * 100 / AVG_WITHOUT))%"
```

## Troubleshooting

### Xdebug Not Connecting

```bash
# Check Xdebug is enabled
hm bash php -v | grep Xdebug

# Check Xdebug configuration
hm bash php --ri xdebug

# Check Xdebug log
hm bash cat /var/log/xdebug.log

# Test connection manually
hm bash php -d xdebug.start_with_request=yes -r "echo 'Testing Xdebug connection';"

# Verify IDE is listening on correct port
netstat -an | grep 9003

# Check Docker networking
docker inspect dockergento_php | grep IPAddress
```

### Port Conflicts

```bash
# Check if port 9003 is in use
lsof -i :9003

# Kill process using port
kill $(lsof -t -i:9003)

# Use alternative port
hm bash php -d xdebug.client_port=9001 bin/magento
```

### Connection Timeout

```bash
# Increase Xdebug timeout
hm bash php -d xdebug.connect_timeout_ms=2000 bin/magento

# Check firewall settings
sudo ufw status

# Allow Xdebug port
sudo ufw allow 9003/tcp
```

## Best Practices

1. **Keep Xdebug disabled by default** (5-10x performance penalty)
2. **Enable only when actively debugging**
3. **Use trigger mode** (start_with_request=trigger)
4. **Configure IDE path mappings** correctly
5. **Use Xdebug 3** (faster than Xdebug 2)
6. **Disable for performance testing** (always)
7. **Close debug sessions** when done
8. **Monitor Xdebug logs** for connection issues
9. **Use profiling mode** for performance analysis
10. **Document IDE setup** for team members

## Common Use Cases

### Debug Magento Plugin

```bash
# Enable Xdebug
hm debug-on

# Set breakpoint in plugin method
# File: app/code/Vendor/Module/Plugin/ProductPlugin.php
# Line: public function beforeSave(...)

# Trigger plugin execution
curl "http://localhost/admin/catalog/product/save/?XDEBUG_SESSION_START=PHPSTORM" \
     -X POST -d "product[sku]=test"

# Disable Xdebug
hm debug-off
```

### Debug REST API

```bash
# Enable Xdebug
hm debug-on

# Start IDE listener

# Call API with debug trigger
curl "http://localhost/rest/V1/products/SKU123?XDEBUG_SESSION_START=PHPSTORM" \
     -H "Authorization: Bearer <token>"

# Debug in IDE

# Disable Xdebug
hm debug-off
```

### Debug CLI Command

```bash
# Enable Xdebug
hm debug-on

# Run command with debug enabled
hm bash php -d xdebug.start_with_request=yes \
            bin/magento customer:hash:upgrade

# Disable Xdebug
hm debug-off
```

## Integration Examples

### Git Hook (Pre-Commit)

```bash
#!/bin/bash
# File: .git/hooks/pre-commit

# Ensure Xdebug is off before committing
if hm bash php -v | grep -q "Xdebug"; then
    echo "WARNING: Xdebug is enabled"
    echo "Run: hm debug-off"
    exit 1
fi
```

### CI/CD Integration

```yaml
# File: .gitlab-ci.yml
test:
  before_script:
    - hm debug-off  # Ensure Xdebug is disabled for tests
  script:
    - hm bash php bin/magento setup:di:compile
    - hm bash php vendor/bin/phpunit
```

## Quality Bar

- Xdebug enables/disables without errors
- Container restarts successfully
- IDE connects to Xdebug properly
- Breakpoints trigger correctly
- Path mappings configured correctly
- Xdebug disabled by default (performance)
- Debug session closes cleanly
- No port conflicts
- Logs show successful connections
- Team documentation updated
