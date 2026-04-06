---
name: dockergento-varnish-controller
description: Manages Varnish cache activation in Dockergento using hm varnish-on and hm varnish-off commands. Use when testing full-page cache behavior, optimizing performance, or simulating production caching.
allowed-tools: Read Grep Glob Bash Edit Write
model: sonnet
---

# Dockergento Varnish Controller

Controls Varnish full-page cache in Hiberus Dockergento environments using `hm varnish-on` and `hm varnish-off` commands for testing production-like caching behavior.

## When to Use

- Testing full-page cache (FPC) functionality
- Simulating production caching behavior
- Performance testing with cache layers
- Debugging cache invalidation issues
- Validating ESI (Edge Side Includes) blocks
- Testing cache purge mechanisms
- Cache-aware development workflow
- Before deploying caching changes

## Requirements

- Hiberus Dockergento environment (https://github.com/hiberus-magento/hiberus-dockergento)
- `hm` CLI tool installed and configured
- Docker containers running
- Varnish container available
- Magento configured for Varnish (app/etc/env.php)

## Execution Workflow

1. **Assess caching need**
   - Identify if Varnish testing required
   - Check current cache state
   - Review Magento Varnish configuration
   - Plan testing scenario

2. **Enable Varnish**
   - Run `hm varnish-on`
   - Wait for routing update
   - Verify Varnish is active
   - Warm up cache if needed

3. **Test caching behavior**
   - Access pages and check cache headers
   - Test cache invalidation
   - Verify ESI blocks work
   - Check cache hit ratios

4. **Disable Varnish**
   - Run `hm varnish-off`
   - Return to direct PHP-FPM access
   - Verify uncached behavior

5. **Analyze results**
   - Review cache logs
   - Check performance metrics
   - Document findings

## Command Reference

### Basic Commands

```bash
# Enable Varnish
hm varnish-on

# Disable Varnish
hm varnish-off

# Check if Varnish is active
curl -I http://localhost | grep -i "X-Varnish\|X-Cache"

# Show Varnish version
hm bash -c varnish varnishd -V

# Varnish stats
hm bash -c varnish varnishstat -1
```

### Verify Varnish Status

```bash
# Check for Varnish headers
curl -I http://localhost | grep -E "X-Varnish|X-Cache|Age"

# Check cache hit/miss
curl -I http://localhost 2>&1 | grep "X-Magento-Cache-Debug"

# Show Varnish backend health
hm bash -c varnish varnishadm backend.list
```

## Implementation Patterns

### Pattern 1: Automated Cache Testing Session

```bash
#!/bin/bash
# Script: test_varnish_cache.sh

set -e

URL="${1:-http://localhost}"

echo "Starting Varnish cache testing session..."

# Enable Varnish
echo "Enabling Varnish..."
hm varnish-on

sleep 3

# Verify Varnish is active
RESPONSE=$(curl -sI "${URL}")
if ! echo "${RESPONSE}" | grep -q "X-Varnish"; then
    echo "ERROR: Varnish not detected"
    exit 1
fi

echo "✓ Varnish enabled successfully"
echo ""

# Test 1: Cache MISS (first request)
echo "Test 1: Initial request (expect MISS)..."
HEADERS=$(curl -sI "${URL}")
echo "${HEADERS}" | grep -E "X-Cache|X-Varnish|Age"
echo ""

# Test 2: Cache HIT (second request)
echo "Test 2: Second request (expect HIT)..."
sleep 1
HEADERS=$(curl -sI "${URL}")
echo "${HEADERS}" | grep -E "X-Cache|X-Varnish|Age"
echo ""

# Test 3: Cache invalidation
echo "Test 3: Cache invalidation..."
hm bash php bin/magento cache:flush full_page
sleep 1

echo "Test 4: After flush (expect MISS)..."
HEADERS=$(curl -sI "${URL}")
echo "${HEADERS}" | grep -E "X-Cache|X-Varnish|Age"
echo ""

# Show Varnish statistics
echo "Varnish Statistics:"
hm bash -c varnish varnishstat -1 | grep -E "cache_hit|cache_miss|n_object"

echo ""
echo "Press Enter to disable Varnish and finish..."
read

# Disable Varnish
echo "Disabling Varnish..."
hm varnish-off

echo "Cache testing completed"
```

### Pattern 2: Performance Comparison

```bash
#!/bin/bash
# Script: compare_cache_performance.sh

set -e

URL="${1:-http://localhost}"
REQUESTS=100

echo "Performance comparison: With/Without Varnish"
echo "URL: ${URL}"
echo "Requests: ${REQUESTS}"
echo ""

# Test WITHOUT Varnish
echo "Testing WITHOUT Varnish..."
hm varnish-off
sleep 3

START=$(date +%s%N)
for i in $(seq 1 ${REQUESTS}); do
    curl -s "${URL}" > /dev/null
done
END=$(date +%s%N)
DURATION_NO_VARNISH=$(( (END - START) / 1000000 ))
AVG_NO_VARNISH=$((DURATION_NO_VARNISH / REQUESTS))

echo "  Total time: ${DURATION_NO_VARNISH}ms"
echo "  Average per request: ${AVG_NO_VARNISH}ms"
echo ""

# Test WITH Varnish
echo "Testing WITH Varnish..."
hm varnish-on
sleep 3

# Warm up cache
curl -s "${URL}" > /dev/null
sleep 1

START=$(date +%s%N)
for i in $(seq 1 ${REQUESTS}); do
    curl -s "${URL}" > /dev/null
done
END=$(date +%s%N)
DURATION_WITH_VARNISH=$(( (END - START) / 1000000 ))
AVG_WITH_VARNISH=$((DURATION_WITH_VARNISH / REQUESTS))

echo "  Total time: ${DURATION_WITH_VARNISH}ms"
echo "  Average per request: ${AVG_WITH_VARNISH}ms"
echo ""

# Results
IMPROVEMENT=$((DURATION_NO_VARNISH - DURATION_WITH_VARNISH))
SPEEDUP=$((DURATION_NO_VARNISH * 100 / DURATION_WITH_VARNISH))

echo "Results:"
echo "  Time saved: ${IMPROVEMENT}ms"
echo "  Speedup: ${SPEEDUP}%"
echo "  Requests/sec without Varnish: $((REQUESTS * 1000 / DURATION_NO_VARNISH))"
echo "  Requests/sec with Varnish: $((REQUESTS * 1000 / DURATION_WITH_VARNISH))"

# Disable Varnish
hm varnish-off
```

### Pattern 3: Cache Warmup Script

```bash
#!/bin/bash
# Script: warmup_varnish_cache.sh

set -e

BASE_URL="${1:-http://localhost}"

echo "Warming up Varnish cache..."

# Enable Varnish
hm varnish-on
sleep 3

# Clear cache first
echo "Clearing cache..."
hm bash php bin/magento cache:flush full_page

# Warmup homepage
echo "Warming up homepage..."
curl -s "${BASE_URL}/" > /dev/null

# Warmup category pages
echo "Warming up category pages..."
CATEGORIES=$(hm bash php bin/magento catalog:category:list | grep -oE "http[s]?://[^\s]+")
for url in ${CATEGORIES}; do
    echo "  ${url}"
    curl -s "${url}" > /dev/null
    sleep 0.5
done

# Warmup product pages
echo "Warming up product pages..."
PRODUCTS=$(hm bash php bin/magento catalog:product:list --limit 20 | grep -oE "http[s]?://[^\s]+")
for url in ${PRODUCTS}; do
    echo "  ${url}"
    curl -s "${url}" > /dev/null
    sleep 0.5
done

# Show cache statistics
echo ""
echo "Cache warmup completed"
echo "Varnish Statistics:"
hm bash -c varnish varnishstat -1 | grep -E "cache_hit|cache_miss|n_object"
```

### Pattern 4: Cache Header Inspector

```bash
#!/bin/bash
# Script: inspect_cache_headers.sh

set -e

URL="${1}"

if [ -z "${URL}" ]; then
    echo "Usage: $0 <url>"
    exit 1
fi

echo "Inspecting cache headers for: ${URL}"
echo ""

# Enable Varnish
hm varnish-on
sleep 2

# First request (MISS)
echo "=== First Request (Cache MISS) ==="
HEADERS=$(curl -sI "${URL}")
echo "${HEADERS}" | grep -E "HTTP|X-Varnish|X-Cache|Age|X-Magento-Cache|Cache-Control"
echo ""

sleep 2

# Second request (HIT)
echo "=== Second Request (Cache HIT) ==="
HEADERS=$(curl -sI "${URL}")
echo "${HEADERS}" | grep -E "HTTP|X-Varnish|X-Cache|Age|X-Magento-Cache|Cache-Control"
echo ""

# Parse key headers
CACHE_STATUS=$(echo "${HEADERS}" | grep "X-Cache:" | awk '{print $2}')
AGE=$(echo "${HEADERS}" | grep "Age:" | awk '{print $2}')
VARNISH=$(echo "${HEADERS}" | grep "X-Varnish:" | awk '{print $2}')

echo "Summary:"
echo "  Cache Status: ${CACHE_STATUS:-Unknown}"
echo "  Age: ${AGE:-0} seconds"
echo "  X-Varnish: ${VARNISH:-N/A}"

if [ "${CACHE_STATUS}" = "HIT" ]; then
    echo "  ✓ Page is cached by Varnish"
else
    echo "  ✗ Page is not cached (check Cache-Control headers)"
fi
```

### Pattern 5: ESI Block Testing

```bash
#!/bin/bash
# Script: test_esi_blocks.sh

set -e

URL="${1:-http://localhost}"

echo "Testing ESI (Edge Side Includes) blocks..."

# Enable Varnish
hm varnish-on
sleep 3

# Check if ESI is enabled in Magento
ESI_ENABLED=$(hm bash php bin/magento config:show system/full_page_cache/esi_enabled)
if [ "${ESI_ENABLED}" != "1" ]; then
    echo "WARNING: ESI not enabled in Magento"
    echo "Enable with: bin/magento config:set system/full_page_cache/esi_enabled 1"
fi

# Fetch page and look for ESI tags
echo "Fetching page and checking for ESI tags..."
CONTENT=$(curl -s "${URL}")

if echo "${CONTENT}" | grep -q "<esi:include"; then
    echo "✓ ESI tags found in response"
    echo "${CONTENT}" | grep -o "<esi:include[^>]*>" | head -5
else
    echo "✗ No ESI tags found"
fi

# Check Varnish ESI processing
echo ""
echo "Varnish ESI Statistics:"
hm bash -c varnish varnishstat -1 | grep -i esi

# Disable Varnish
hm varnish-off
```

## Magento Configuration

### Check Varnish Configuration

```bash
# Check if Varnish is configured
hm bash php bin/magento config:show system/full_page_cache/caching_application

# Expected output: 2 (Varnish)

# Check Varnish servers
hm bash php bin/magento config:show system/full_page_cache/varnish/backend_host
hm bash php bin/magento config:show system/full_page_cache/varnish/backend_port

# Generate VCL file
hm bash php bin/magento varnish:vcl:generate > varnish.vcl
```

### Enable Varnish in Magento

```bash
# Set Varnish as FPC
hm bash php bin/magento config:set system/full_page_cache/caching_application 2

# Configure Varnish backend
hm bash php bin/magento config:set system/full_page_cache/varnish/backend_host nginx
hm bash php bin/magento config:set system/full_page_cache/varnish/backend_port 80

# Configure Varnish servers (for purge)
hm bash php bin/magento config:set system/full_page_cache/varnish/access_list localhost

# Enable ESI
hm bash php bin/magento config:set system/full_page_cache/esi_enabled 1

# Flush config cache
hm bash php bin/magento cache:flush config
```

## Cache Debugging

### Check Cache Hit Ratio

```bash
#!/bin/bash
# Script: cache_hit_ratio.sh

hm bash -c varnish varnishstat -1 | awk '
/cache_hit/ { hit=$2 }
/cache_miss/ { miss=$2 }
END {
    total = hit + miss
    if (total > 0) {
        ratio = (hit / total) * 100
        printf "Cache Hits: %d\n", hit
        printf "Cache Misses: %d\n", miss
        printf "Hit Ratio: %.2f%%\n", ratio
    } else {
        print "No cache statistics available"
    }
}'
```

### Monitor Varnish Log

```bash
# Live log of Varnish requests
hm bash -c varnish varnishlog

# Filter for specific URL
hm bash -c varnish varnishlog -q "ReqURL ~ '/catalog/product/view/'"

# Show only cache hits
hm bash -c varnish varnishlog -q "VCL_call eq HIT"

# Show only cache misses
hm bash -c varnish varnishlog -q "VCL_call eq MISS"
```

### Purge Varnish Cache

```bash
# Purge specific URL
curl -X PURGE http://localhost/specific-page

# Purge all cache via Magento
hm bash php bin/magento cache:flush full_page

# Purge all Varnish cache (nuclear option)
hm bash -c varnish varnishadm "ban req.url ~ /"
```

## Testing Scenarios

### Test 1: Homepage Caching

```bash
# Enable Varnish
hm varnish-on

# First request (MISS)
curl -sI http://localhost | grep "X-Cache"
# Expected: MISS

# Second request (HIT)
curl -sI http://localhost | grep "X-Cache"
# Expected: HIT

# Disable Varnish
hm varnish-off
```

### Test 2: Category Page Caching

```bash
# Enable Varnish
hm varnish-on

# Access category page
CATEGORY_URL="http://localhost/women/tops-women.html"

# First request
curl -sI "${CATEGORY_URL}" | grep "X-Cache"

# Second request (should be cached)
curl -sI "${CATEGORY_URL}" | grep "X-Cache"

# Disable Varnish
hm varnish-off
```

### Test 3: Cache Invalidation

```bash
# Enable Varnish
hm varnish-on

# Warm cache
curl -s http://localhost > /dev/null

# Verify cache hit
curl -sI http://localhost | grep "X-Cache: HIT"

# Invalidate cache
hm bash php bin/magento cache:flush full_page

# Verify cache miss
curl -sI http://localhost | grep "X-Cache: MISS"

# Disable Varnish
hm varnish-off
```

### Test 4: ESI Block Caching

```bash
# Enable Varnish
hm varnish-on

# Enable ESI
hm bash php bin/magento config:set system/full_page_cache/esi_enabled 1
hm bash php bin/magento cache:flush config

# Check for ESI tags in response
curl -s http://localhost | grep "esi:include"

# Disable Varnish
hm varnish-off
```

## Troubleshooting

### Varnish Not Caching Pages

```bash
# Check Varnish configuration
hm bash php bin/magento config:show system/full_page_cache/caching_application

# Should be "2" (Varnish)
# If not, set it:
hm bash php bin/magento config:set system/full_page_cache/caching_application 2

# Check Cache-Control headers
curl -sI http://localhost | grep "Cache-Control"

# Check for no-cache directives
curl -sI http://localhost | grep -E "no-cache|no-store|private"
```

### Cache Not Invalidating

```bash
# Check Varnish purge configuration
hm bash php bin/magento config:show system/full_page_cache/varnish/access_list

# Test manual purge
curl -X PURGE http://localhost/

# Check Varnish ban list
hm bash -c varnish varnishadm ban.list

# Force clear all cache
hm bash -c varnish varnishadm "ban req.url ~ /"
```

### ESI Not Working

```bash
# Verify ESI enabled in Magento
hm bash php bin/magento config:show system/full_page_cache/esi_enabled

# Enable if needed
hm bash php bin/magento config:set system/full_page_cache/esi_enabled 1

# Check Varnish VCL for ESI
hm bash -c varnish cat /etc/varnish/default.vcl | grep esi

# Generate correct VCL
hm bash php bin/magento varnish:vcl:generate --export-version=6 > varnish6.vcl
```

## Best Practices

1. **Keep Varnish disabled during active development** (cache confusion)
2. **Enable for testing cache-related features**
3. **Use cache warmup scripts** after enabling
4. **Monitor cache hit ratios** (target >90%)
5. **Test cache invalidation** thoroughly
6. **Configure appropriate TTL** values
7. **Use ESI for dynamic blocks** (cart, customer menu)
8. **Purge cache selectively** (avoid full purges)
9. **Test before production deployment**
10. **Document caching behavior** for team

## Common Pitfalls

### ❌ Don't

- Don't leave Varnish on during development (stale cache)
- Don't deploy Varnish changes without testing
- Don't ignore cache headers (Cache-Control)
- Don't use GET parameters in cached URLs
- Don't cache personalized content (use ESI)
- Don't forget to configure purge ACL

### ✅ Do

- Do disable Varnish for development
- Do test cache invalidation scenarios
- Do use appropriate Cache-Control headers
- Do implement proper ESI blocks
- Do monitor cache statistics
- Do configure grace mode for resilience

## Integration Examples

### Development Workflow

```bash
#!/bin/bash
# File: dev_workflow.sh

# Development mode - Varnish OFF
echo "Development mode: Varnish disabled"
hm varnish-off

# Code changes...

# Testing mode - Varnish ON
echo "Testing cache behavior..."
hm varnish-on
./scripts/test_varnish_cache.sh

# Back to development
hm varnish-off
```

### CI/CD Integration

```yaml
# File: .gitlab-ci.yml
test-cache:
  script:
    - hm varnish-on
    - php bin/magento cache:flush
    - ./tests/cache_test.sh
    - hm varnish-off
```

## Quality Bar

- Varnish enables/disables without errors
- Magento detects Varnish correctly
- Cache headers present in responses
- Cache hit ratio > 90% for static pages
- Cache invalidation works correctly
- ESI blocks function properly
- No cache confusion during development
- Performance improvement measurable
- Cache purge ACL configured correctly
- Team understands when to enable/disable
