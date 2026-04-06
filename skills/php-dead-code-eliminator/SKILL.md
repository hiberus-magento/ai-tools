---
name: php-dead-code-eliminator
description: Analyzes PHP call graph to identify and eliminate unreachable variables, properties, methods, and parameters. Use when refactoring legacy code, reducing technical debt, or optimizing codebase maintenance surface.
allowed-tools: Read Grep Glob Bash
model: sonnet
---

# PHP Dead Code Eliminator

Analyzes the PHP call graph to identify and safely eliminate unreachable code elements: unused variables, properties, methods, and parameters. This reduces the maintenance surface and improves code clarity.

## When to Use

- Refactoring legacy PHP codebases
- After feature removal or major refactoring
- Before starting new development in old modules
- When reducing technical debt
- When improving code maintainability

## Execution Workflow

1. **Analyze the codebase structure**
   - Use Glob to find all PHP files
   - Identify entry points (controllers, commands, public APIs)
   - Map class hierarchy and dependencies

2. **Build call graph**
   - Parse all class methods and functions
   - Identify method calls, property accesses, and variable usage
   - Track inheritance and trait usage
   - Map interface implementations

3. **Identify unreachable code**
   - Start from entry points (controllers, CLI commands, public APIs)
   - Mark all reachable methods, properties, and functions
   - Flag unmarked elements as potentially dead code
   - Verify with static analysis patterns

4. **Categorize findings**
   - **High confidence**: Private methods never called, unused properties
   - **Medium confidence**: Protected methods not overridden or called
   - **Low confidence**: Public methods (may be called dynamically)
   - **Variables**: Assigned but never read

5. **Safety checks**
   - Check for magic method usage (__call, __get, __set)
   - Verify not used in Reflection API calls
   - Look for dynamic method/property access patterns
   - Check for serialization/deserialization usage

6. **Generate refactoring report**
   - List all dead code candidates by confidence level
   - Provide file paths and line numbers
   - Suggest safe removal order (start with private members)
   - Flag risky removals requiring manual review

7. **Apply safe removals** (if requested)
   - Start with high-confidence candidates
   - Remove unused private methods and properties
   - Remove unused parameters (replace with void)
   - Remove dead variables

## Analysis Patterns

### Unreachable Methods
```php
class Example {
    public function used() {
        // Called from controllers
    }
    
    private function unused() {
        // Never called - HIGH CONFIDENCE dead code
    }
    
    protected function maybeUnused() {
        // Not called in codebase, not overridden - MEDIUM CONFIDENCE
    }
}
```

### Unused Properties
```php
class Example {
    private $usedProperty;
    private $unusedProperty; // Never accessed - HIGH CONFIDENCE
    
    public function __construct() {
        $this->usedProperty = 'value';
        $this->unusedProperty = 'value'; // Assigned but never read
    }
}
```

### Dead Variables
```php
function example() {
    $used = getData();
    $unused = getOtherData(); // Assigned but never referenced
    
    return $used;
}
```

### Unused Parameters
```php
function process($required, $unused) {
    // $unused parameter is never referenced in function body
    return $required;
}
```

## Safety Considerations

### ⚠️ Do NOT remove if:
- Used in magic methods (__call, __get, __set, __isset)
- Accessed via Reflection API
- Used in dynamic property/method access: `$obj->$propertyName`
- Part of serialization/unserialization
- Required by parent class or interface
- Used in eval() or variable functions
- Documented as public API (even if not used internally)

### ✅ Safe to remove:
- Private methods never called
- Private properties never accessed
- Variables assigned but never read
- Parameters never referenced in function body
- After verifying no dynamic access patterns

## Tools Used

- **Rector**: For automated dead code removal
- **PHPStan**: For static analysis verification
- **nikic/php-parser**: For AST-based analysis
- **Grep**: For pattern matching and verification

## Output Format

Generate a markdown report: `dead-code-analysis.md`

```markdown
# Dead Code Analysis Report

**Date:** YYYY-MM-DD
**Files Analyzed:** N
**Total Dead Code Candidates:** N

## Summary

- High Confidence: N items (safe to remove)
- Medium Confidence: N items (review recommended)
- Low Confidence: N items (manual review required)

## High Confidence Dead Code

### Unused Private Methods

**File:** src/Module/Service.php
**Line:** 145-152
**Method:** `Service::unusedPrivateMethod()`
**Reason:** Never called, no dynamic access patterns
**Action:** Safe to remove

### Unused Private Properties

**File:** src/Module/Model.php
**Line:** 23
**Property:** `Model::$unusedProperty`
**Reason:** Assigned but never read
**Action:** Safe to remove

## Medium Confidence Dead Code

[List items requiring review]

## Low Confidence Dead Code

[List items requiring manual review]

## Rector Rules to Apply

```php
// rector.php
use Rector\DeadCode\Rector\ClassMethod\RemoveUnusedPrivateMethodRector;
use Rector\DeadCode\Rector\Property\RemoveUnusedPrivatePropertyRector;
// ... additional rules
```

## Next Steps

1. Review high confidence items
2. Apply Rector rules for safe removal
3. Run tests after each removal batch
4. Manually review medium/low confidence items
```

## Rector Integration

When applying automatic refactoring:

```bash
# Install Rector
composer require rector/rector --dev

# Apply dead code removal rules
vendor/bin/rector process src/ --config=rector-dead-code.php --dry-run

# Apply changes after review
vendor/bin/rector process src/ --config=rector-dead-code.php
```

## Best Practices

1. **Start small**: Remove high-confidence items first
2. **Test continuously**: Run tests after each removal batch
3. **Review manually**: Always review generated changes before applying
4. **Commit frequently**: One category of removal per commit
5. **Monitor production**: Watch for errors after deployment
6. **Document findings**: Keep report for future reference

## Example Usage

**User request:**
```
Analyze this PHP module for dead code and remove safe candidates
```

**Claude execution:**
1. Scans all PHP files in current directory
2. Builds call graph starting from public entry points
3. Identifies unreachable code with confidence levels
4. Generates detailed report with file/line references
5. If approved, applies safe removals using Rector
6. Runs tests to verify no breakage

## Quality Bar

- No false positives in high-confidence category
- All findings include file path and line number
- Every suggestion includes safety rationale
- Manual review required before any removal
- Test suite must pass after refactoring
