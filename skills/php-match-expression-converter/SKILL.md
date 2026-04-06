---
name: php-match-expression-converter
description: Converts complex switch statements and chained if-else blocks into concise match expressions with strict return types. Use when modernizing PHP 8+ code or improving code clarity.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Match Expression Converter

Transforms verbose switch statements and chained if-else blocks into PHP 8.0+ match expressions, providing strict type comparison, exhaustiveness checking, and concise return-focused syntax.

## When to Use

- Upgrading to PHP 8.0+
- Refactoring switch statements that return values
- Simplifying conditional logic
- Improving type safety (strict comparison)
- When working with enums
- Reducing code verbosity

## Match vs Switch Differences

| Feature | switch | match |
|---------|--------|-------|
| Comparison | Loose (`==`) | Strict (`===`) |
| Return value | Manual assignment | Expression |
| Fall-through | Explicit break needed | No fall-through |
| Exhaustiveness | No check | PHPStan/IDE checks |
| Multiple conditions | Manual or fall-through | Comma-separated |
| Default | `default:` | `default =>` |

## Execution Workflow

1. **Find refactoring candidates**
   - Grep for switch statements
   - Find if-elseif-else chains
   - Locate value-returning conditionals
   - Identify enum-based switches

2. **Analyze convertibility**
   - Check if returns single value
   - Verify no fall-through logic needed
   - Ensure strict comparison is safe
   - Assess readability improvement

3. **Design match expression**
   - Extract condition cases
   - Combine multiple conditions with comma
   - Add default case if needed
   - Consider multi-line formatting

4. **Convert to match**
   - Replace switch with match
   - Remove break statements
   - Convert assignments to returns
   - Handle exceptions if needed

5. **Verify correctness**
   - Test with all cases
   - Verify strict comparison behavior
   - Check exhaustiveness with static analysis
   - Ensure same output as before

## Refactoring Patterns

### Pattern 1: Simple Switch to Match

**Before**:
```php
function getStatusLabel(string $status): string {
    switch ($status) {
        case 'pending':
            return 'Pending Approval';
        case 'approved':
            return 'Approved';
        case 'rejected':
            return 'Rejected';
        default:
            return 'Unknown';
    }
}
```

**After**:
```php
function getStatusLabel(string $status): string {
    return match ($status) {
        'pending' => 'Pending Approval',
        'approved' => 'Approved',
        'rejected' => 'Rejected',
        default => 'Unknown',
    };
}
```

### Pattern 2: Switch with Multiple Cases

**Before**:
```php
function getHttpStatusCategory(int $code): string {
    switch ($code) {
        case 200:
        case 201:
        case 204:
            return 'Success';
        case 400:
        case 404:
        case 422:
            return 'Client Error';
        case 500:
        case 502:
        case 503:
            return 'Server Error';
        default:
            return 'Other';
    }
}
```

**After**:
```php
function getHttpStatusCategory(int $code): string {
    return match ($code) {
        200, 201, 204 => 'Success',
        400, 404, 422 => 'Client Error',
        500, 502, 503 => 'Server Error',
        default => 'Other',
    };
}
```

### Pattern 3: If-Elseif Chain to Match

**Before**:
```php
function calculateShipping(string $method): float {
    if ($method === 'standard') {
        return 5.99;
    } elseif ($method === 'express') {
        return 12.99;
    } elseif ($method === 'overnight') {
        return 24.99;
    } else {
        throw new \InvalidArgumentException('Unknown shipping method');
    }
}
```

**After**:
```php
function calculateShipping(string $method): float {
    return match ($method) {
        'standard' => 5.99,
        'express' => 12.99,
        'overnight' => 24.99,
        default => throw new \InvalidArgumentException('Unknown shipping method'),
    };
}
```

### Pattern 4: Enum-Based Match (PHP 8.1+)

**Before**:
```php
enum Priority {
    case Low;
    case Medium;
    case High;
    case Critical;
}

function getColor(Priority $priority): string {
    switch ($priority) {
        case Priority::Low:
            return '#28a745';
        case Priority::Medium:
            return '#ffc107';
        case Priority::High:
            return '#fd7e14';
        case Priority::Critical:
            return '#dc3545';
    }
    // Missing default - no compile error with switch
}
```

**After**:
```php
function getColor(Priority $priority): string {
    return match ($priority) {
        Priority::Low => '#28a745',
        Priority::Medium => '#ffc107',
        Priority::High => '#fd7e14',
        Priority::Critical => '#dc3545',
        // PHPStan/IDE will error if a case is missing!
    };
}
```

### Pattern 5: Match with Complex Expressions

**Before**:
```php
function processPayment(string $method, float $amount): array {
    switch ($method) {
        case 'card':
            $fee = $amount * 0.029 + 0.30;
            $processor = 'stripe';
            break;
        case 'paypal':
            $fee = $amount * 0.034 + 0.35;
            $processor = 'paypal';
            break;
        case 'bank':
            $fee = 1.00;
            $processor = 'plaid';
            break;
        default:
            throw new \Exception('Unknown method');
    }
    
    return [
        'processor' => $processor,
        'fee' => $fee,
        'total' => $amount + $fee,
    ];
}
```

**After**:
```php
function processPayment(string $method, float $amount): array {
    [$processor, $fee] = match ($method) {
        'card' => ['stripe', $amount * 0.029 + 0.30],
        'paypal' => ['paypal', $amount * 0.034 + 0.35],
        'bank' => ['plaid', 1.00],
        default => throw new \Exception('Unknown method'),
    };
    
    return [
        'processor' => $processor,
        'fee' => $fee,
        'total' => $amount + $fee,
    ];
}
```

### Pattern 6: Match with Method Calls

**Before**:
```php
function handleAction(string $action, array $data): void {
    switch ($action) {
        case 'create':
            $this->create($data);
            break;
        case 'update':
            $this->update($data);
            break;
        case 'delete':
            $this->delete($data);
            break;
        default:
            throw new \Exception('Unknown action');
    }
}
```

**After**:
```php
function handleAction(string $action, array $data): void {
    match ($action) {
        'create' => $this->create($data),
        'update' => $this->update($data),
        'delete' => $this->delete($data),
        default => throw new \Exception('Unknown action'),
    };
}
```

## Advanced Match Patterns

### Conditional Expressions
```php
// Type-based matching
$result = match (true) {
    $value instanceof User => $value->getName(),
    $value instanceof Group => $value->getTitle(),
    is_string($value) => $value,
    default => throw new \TypeError('Unsupported type'),
};

// Range matching
$category = match (true) {
    $age < 13 => 'child',
    $age < 18 => 'teen',
    $age < 65 => 'adult',
    default => 'senior',
};
```

### Throwing Exceptions
```php
// match can throw on specific cases
$value = match ($status) {
    'success' => $this->getData(),
    'error' => throw new \RuntimeException('Failed'),
    default => null,
};
```

### Non-Empty Match (No Default)
```php
// Exhaustive enum matching - no default needed
enum Status { case Active; case Inactive; }

$label = match ($status) {
    Status::Active => 'Active',
    Status::Inactive => 'Inactive',
}; // PHPStan ensures all cases covered
```

## When NOT to Use Match

❌ **Don't use match when:**
- Fall-through behavior is needed
- Multiple statements per case (use switch or refactor)
- Loose comparison is required (`==` not `===`)
- Side effects without return value (use switch or if)
- Case conditions have complex logic

✅ **DO use match when:**
- Returning a value based on condition
- Strict comparison is safe/preferred
- Multiple conditions per arm
- Working with enums
- Want exhaustiveness checking
- Improving readability

## Strict Comparison Gotcha

**Before** (loose comparison):
```php
switch ($value) {
    case 0:
        return 'zero';
    case '0':
        return 'string zero';
}

// With $value = '0' returns 'zero' (loose comparison)
```

**After** (strict comparison):
```php
match ($value) {
    0 => 'zero',
    '0' => 'string zero',
}

// With $value = '0' returns 'string zero' (strict comparison)
```

## Output Format

```markdown
# Match Expression Conversion Report

**Date:** YYYY-MM-DD
**Switch Statements Found:** N
**If-Else Chains Found:** N
**Conversion Candidates:** N

## High Priority Conversions

### Candidate 1: Status Label Mapping

**File:** src/Service/OrderService.php
**Line:** 45-58
**Type:** Simple value return
**Complexity Reduction:** 13 lines → 6 lines

**Before:**
```php
switch ($status) {
    case 'pending':
        return 'Pending';
    case 'processing':
        return 'Processing';
    case 'complete':
        return 'Complete';
    default:
        return 'Unknown';
}
```

**After:**
```php
return match ($status) {
    'pending' => 'Pending',
    'processing' => 'Processing',
    'complete' => 'Complete',
    default => 'Unknown',
};
```

**Benefits:**
- 54% less code
- Strict comparison (`===`)
- Expression-based (direct return)
- No break statements needed

## Medium Priority Conversions

[Similar structure for candidates with minor complexity]

## Not Convertible

### Example: Multi-Statement Cases

**File:** src/Service/Complex.php
**Line:** 78-95
**Reason:** Multiple statements per case

```php
switch ($action) {
    case 'process':
        $this->validate($data);
        $this->process($data);
        $this->notify();
        break;
    // ...
}
```

**Recommendation:** Refactor to extract methods, then convert

## Summary

- Conversions: 23 eligible
- Lines saved: ~150 (avg 6.5 per conversion)
- Type safety: Strict comparison on all
- Readability: Improved on 20/23 cases
```

## Best Practices

1. **Use for value returns**: Match is expression-based
2. **Leverage strict comparison**: Catches type mismatches
3. **Prefer readability**: Multi-line formatting for complex cases
4. **Check exhaustiveness**: Use PHPStan/Psalm with enums
5. **Extract complex logic**: Keep match arms simple
6. **Use trailing comma**: In multi-line matches
7. **Throw in default**: When invalid input is error

## Example Usage

**User request:**
```
This switch statement looks verbose, can you simplify it?
```

**Claude execution:**
1. Reads the switch statement
2. Verifies it returns a single value
3. Checks for fall-through logic
4. Converts to match expression
5. Tests with all cases
6. Verifies strict comparison is safe

## Quality Bar

- All conversions preserve exact behavior (considering strict comparison)
- Code becomes more concise
- Type safety improved
- Static analysis passes
- Tests verify all cases work
- Exhaustiveness checked for enums
