---
name: php-generator-memory-optimizer
description: Detects loops loading massive datasets into memory and refactors them using yield keyword for efficient iterative processing. Use when facing memory issues, processing large datasets, or optimizing resource-intensive operations.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Generator Memory Optimizer

Identifies loops and methods that load large datasets entirely into memory and refactors them to use PHP generators (`yield`) for memory-efficient lazy evaluation. This prevents memory exhaustion and improves performance when processing large data volumes.

## When to Use

- Memory limit errors during data processing
- Processing large CSV/JSON/XML files
- Iterating over database result sets
- Large array operations and transformations
- Batch processing jobs
- ETL (Extract, Transform, Load) operations
- API responses with pagination

## Execution Workflow

1. **Identify memory-intensive patterns**
   - Grep for large array builders: `$array[] =`, `array_merge`, `array_push`
   - Find database query loops without chunking
   - Locate file reading operations loading entire files
   - Identify array_map/array_filter on large datasets

2. **Analyze memory impact**
   - Estimate data volume processed
   - Check if result is iterated only once
   - Verify if random access is needed
   - Assess current memory usage patterns

3. **Categorize refactoring candidates**
   - **High priority**: OOM errors, multi-GB datasets
   - **Medium priority**: Large but manageable datasets
   - **Low priority**: Small datasets (< 1000 items)

4. **Design generator solution**
   - Convert array-building loops to yield statements
   - Implement chunked processing for database queries
   - Use streaming for file operations
   - Chain generators for multi-step transformations

5. **Refactor code**
   - Replace array builders with generators
   - Update type hints (array → iterable/Generator)
   - Modify consumers to use foreach instead of array functions
   - Add generator chaining where appropriate

6. **Verify correctness**
   - Ensure lazy evaluation doesn't break logic
   - Check that generators are only iterated when needed
   - Verify no random access to results
   - Test with real data volumes

7. **Measure improvement**
   - Compare memory usage before/after
   - Verify performance improvements
   - Document memory savings

## Refactoring Patterns

### Pattern 1: Array Builder Loop

**Before** (loads all into memory):
```php
function getUsers(): array {
    $users = [];
    $result = $db->query("SELECT * FROM users"); // 100K records
    
    while ($row = $result->fetch()) {
        $users[] = new User($row); // Memory: 100K * sizeof(User)
    }
    
    return $users;
}

// Usage
foreach (getUsers() as $user) {
    echo $user->getName();
}
```

**After** (lazy evaluation):
```php
function getUsers(): Generator {
    $result = $db->query("SELECT * FROM users"); // 100K records
    
    while ($row = $result->fetch()) {
        yield new User($row); // Memory: 1 * sizeof(User) at a time
    }
}

// Usage (identical)
foreach (getUsers() as $user) {
    echo $user->getName();
}
```

### Pattern 2: File Processing

**Before**:
```php
function processLogFile(string $path): array {
    $lines = file($path); // Loads entire 2GB file into memory
    $errors = [];
    
    foreach ($lines as $line) {
        if (str_contains($line, 'ERROR')) {
            $errors[] = $line;
        }
    }
    
    return $errors;
}
```

**After**:
```php
function processLogFile(string $path): Generator {
    $handle = fopen($path, 'r');
    
    while (($line = fgets($handle)) !== false) {
        if (str_contains($line, 'ERROR')) {
            yield $line; // Memory: constant, regardless of file size
        }
    }
    
    fclose($handle);
}
```

### Pattern 3: Data Transformation Pipeline

**Before**:
```php
function processOrders(): array {
    $orders = $db->query("SELECT * FROM orders")->fetchAll(); // All orders in memory
    
    $filtered = array_filter($orders, fn($o) => $o['status'] === 'pending');
    $mapped = array_map(fn($o) => new Order($o), $filtered);
    $sorted = usort($mapped, fn($a, $b) => $a->getDate() <=> $b->getDate());
    
    return $mapped;
}
```

**After** (generator chaining):
```php
function fetchOrders(): Generator {
    $result = $db->query("SELECT * FROM orders WHERE status = 'pending' ORDER BY date");
    
    while ($row = $result->fetch()) {
        yield new Order($row);
    }
}

function processOrders(): Generator {
    yield from fetchOrders(); // Chain generators
}
```

### Pattern 4: Chunked Processing

**Before**:
```php
function exportProducts(): array {
    $products = $db->query("SELECT * FROM products")->fetchAll(); // 500K products
    
    $csv = [];
    foreach ($products as $product) {
        $csv[] = implode(',', $product);
    }
    
    return $csv;
}
```

**After**:
```php
function exportProducts(): Generator {
    $offset = 0;
    $limit = 1000;
    
    while (true) {
        $chunk = $db->query(
            "SELECT * FROM products LIMIT $limit OFFSET $offset"
        )->fetchAll();
        
        if (empty($chunk)) {
            break;
        }
        
        foreach ($chunk as $product) {
            yield implode(',', $product);
        }
        
        $offset += $limit;
    }
}
```

## Type Hint Updates

When refactoring to generators, update type hints:

```php
// Before
function getData(): array { /* ... */ }

// After - choose appropriate type
function getData(): Generator { /* ... */ }  // Specific
function getData(): iterable { /* ... */ }   // More flexible (array|Traversable)
function getData(): \Iterator { /* ... */ }  // Interface (less common for yield)
```

## When NOT to Use Generators

❌ **Don't use generators when:**
- Need random access: `$items[5]`
- Need count: `count($items)`
- Need to iterate multiple times
- Working with small datasets (< 1000 items)
- Need array functions: `array_sum()`, `array_unique()`, etc.
- Need to pass to functions expecting arrays

✅ **DO use generators when:**
- Single-pass iteration
- Large datasets (> 10K items)
- Memory constraints
- Streaming data processing
- Lazy evaluation benefits
- Infinite sequences

## Safety Checks

Before refactoring, verify:

1. **Single iteration**: Result is consumed only once
2. **No count needed**: `count()` not called on result
3. **No array access**: No `$result[0]` style access
4. **No array functions**: No `array_map`, `array_filter`, `in_array` on result
5. **No multiple passes**: Not iterated in multiple foreach loops

If any check fails, either:
- Keep array-based approach for that use case
- Convert generator to array at consumption: `iterator_to_array($generator)`
- Refactor consumer to work with generators

## Output Format

Generate a markdown report: `generator-optimization-report.md`

```markdown
# Generator Memory Optimization Report

**Date:** YYYY-MM-DD
**Files Analyzed:** N
**Refactoring Candidates:** N

## Summary

- High Priority (OOM risk): N candidates
- Medium Priority (Large datasets): N candidates  
- Low Priority (Optimization): N candidates
- Estimated Memory Savings: X GB

## High Priority Candidates

### Candidate 1: User Export

**File:** src/Export/UserExporter.php
**Line:** 45-67
**Current Pattern:** Loads all users into array (500K records = ~2GB)
**Memory Impact:** HIGH (causes OOM on production)
**Refactoring:** Convert to generator with chunked queries

**Before:**
```php
public function export(): array {
    return $this->db->query("SELECT * FROM users")->fetchAll();
}
```

**After:**
```php
public function export(): Generator {
    $offset = 0;
    $limit = 1000;
    while ($chunk = $this->db->query("...")->fetchAll()) {
        yield from $chunk;
        $offset += $limit;
    }
}
```

**Memory Savings:** ~2GB → ~10MB (99.5% reduction)

## Medium Priority Candidates

[Similar structure for medium priority items]

## Refactoring Checklist

- [ ] Update method return types (array → Generator/iterable)
- [ ] Convert array builders to yield statements
- [ ] Implement chunked database queries
- [ ] Update consumers to use foreach (if needed)
- [ ] Run memory profiling tests
- [ ] Update PHPDoc blocks
- [ ] Verify no random access requirements

## Performance Impact

| Operation | Before (Memory) | After (Memory) | Savings |
|-----------|----------------|----------------|---------|
| User export | 2.1 GB | 8 MB | 99.6% |
| Log processing | 1.5 GB | 2 MB | 99.8% |
| Order batch | 800 MB | 5 MB | 99.4% |
| **Total** | **4.4 GB** | **15 MB** | **99.7%** |

## Next Steps

1. Review and approve refactorings
2. Update tests to work with generators
3. Apply refactorings one file at a time
4. Run memory profiling before/after
5. Monitor production after deployment
```

## Testing Generators

```php
// Test generator produces expected values
public function testGeneratorOutput(): void {
    $generator = $this->service->getData();
    $result = iterator_to_array($generator);
    
    $this->assertCount(3, $result);
    $this->assertEquals('expected', $result[0]);
}

// Test generator handles empty results
public function testEmptyGenerator(): void {
    $generator = $this->service->getEmptyData();
    $result = iterator_to_array($generator);
    
    $this->assertEmpty($result);
}

// Test memory usage
public function testMemoryEfficiency(): void {
    $memoryBefore = memory_get_usage();
    
    foreach ($this->service->getLargeDataset() as $item) {
        // Process item
    }
    
    $memoryAfter = memory_get_usage();
    $memoryUsed = $memoryAfter - $memoryBefore;
    
    $this->assertLessThan(10 * 1024 * 1024, $memoryUsed); // < 10MB
}
```

## Best Practices

1. **Document generator behavior**: Add PHPDoc with `@return Generator<KeyType, ValueType>`
2. **Use generator delegation**: `yield from` for chaining generators
3. **Close resources**: Use try-finally to close file handles
4. **Type hint as iterable**: More flexible than Generator
5. **Test with real volumes**: Verify memory savings with production-like data
6. **Profile before/after**: Use Xdebug or Blackfire for measurements
7. **Keep consumers simple**: foreach is ideal for generators

## Example Usage

**User request:**
```
This script runs out of memory when processing all orders, can you optimize it?
```

**Claude execution:**
1. Reads the problematic code
2. Identifies array-building pattern loading all orders
3. Proposes generator-based refactoring
4. Shows before/after memory usage
5. Implements the refactoring
6. Updates tests to work with generators
7. Verifies memory usage improvements

## Quality Bar

- All refactorings preserve exact behavior
- Memory usage verified with profiling
- No breaking changes to public APIs (type hints remain compatible)
- Tests updated and passing
- Documentation updated with memory characteristics
