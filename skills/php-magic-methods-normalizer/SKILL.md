---
name: php-magic-methods-normalizer
description: Replaces unpredictable magic methods (__get, __set, __isset, __unset) with explicit getters and setters for type inference and IDE support. Use when improving code maintainability, enabling static analysis, or refactoring legacy OOP structures.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Magic Methods Normalizer

Converts magic method implementations (`__get`, `__set`, `__isset`, `__unset`) into explicit, typed getter and setter methods. This improves type safety, enables IDE autocomplete, allows static analysis, and makes code behavior predictable.

## When to Use

- Refactoring legacy PHP code with magic methods
- Improving IDE support and autocomplete
- Enabling PHPStan/Psalm static analysis
- Making implicit behavior explicit
- Improving code maintainability
- Before adding strict typing to a codebase
- When onboarding new developers to legacy code

## Execution Workflow

1. **Find classes using magic methods**
   - Grep for `__get`, `__set`, `__isset`, `__unset`
   - Identify property storage mechanisms (arrays, stdClass)
   - Map which properties are accessed via magic methods

2. **Analyze magic method usage**
   - Track property access patterns across codebase
   - Identify property types from usage context
   - Determine read-only vs read-write properties
   - Check for property existence validation

3. **Design explicit interface**
   - Generate getter/setter method signatures
   - Infer types from usage patterns
   - Decide on naming convention (get/set, is/has for booleans)
   - Plan migration path (keep magic methods during transition)

4. **Generate explicit methods**
   - Create typed getter methods
   - Create typed setter methods with validation
   - Add fluent interfaces where appropriate (return $this)
   - Include PHPDoc for complex types

5. **Refactor consumers**
   - Replace `$obj->property` with `$obj->getProperty()`
   - Replace `$obj->property = $value` with `$obj->setProperty($value)`
   - Update `isset($obj->property)` with `$obj->hasProperty()`
   - Update `unset($obj->property)` with `$obj->clearProperty()`

6. **Deprecate magic methods**
   - Add @deprecated annotation
   - Log usage in magic methods (for migration tracking)
   - Keep magic methods as fallback during transition
   - Plan removal timeline

7. **Remove magic methods** (final step)
   - Verify no usage remains in codebase
   - Remove `__get`, `__set`, `__isset`, `__unset`
   - Update tests
   - Run static analysis to confirm

## Refactoring Patterns

### Pattern 1: Basic Magic Property Access

**Before** (magic methods):
```php
class User {
    private array $data = [];
    
    public function __get(string $name) {
        return $this->data[$name] ?? null;
    }
    
    public function __set(string $name, $value): void {
        $this->data[$name] = $value;
    }
    
    public function __isset(string $name): bool {
        return isset($this->data[$name]);
    }
}

// Usage (no IDE support, no type safety)
$user = new User();
$user->name = 'John';      // No autocomplete
$user->email = 'john@example.com';
echo $user->name;           // IDE doesn't know this exists
```

**After** (explicit methods):
```php
class User {
    private array $data = [];
    
    public function getName(): ?string {
        return $this->data['name'] ?? null;
    }
    
    public function setName(string $name): self {
        $this->data['name'] = $name;
        return $this;
    }
    
    public function getEmail(): ?string {
        return $this->data['email'] ?? null;
    }
    
    public function setEmail(string $email): self {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException('Invalid email');
        }
        $this->data['email'] = $email;
        return $this;
    }
    
    public function hasName(): bool {
        return isset($this->data['name']);
    }
    
    // Keep magic methods during transition (deprecated)
    /** @deprecated Use explicit getters/setters */
    public function __get(string $name) {
        trigger_error("Magic __get is deprecated, use get" . ucfirst($name) . "()", E_USER_DEPRECATED);
        return $this->data[$name] ?? null;
    }
}

// Usage (full IDE support, type safety)
$user = new User();
$user->setName('John')     // Autocomplete works
     ->setEmail('john@example.com'); // Fluent interface
echo $user->getName();      // IDE knows return type
```

### Pattern 2: Type-Safe Property Access

**Before**:
```php
class Product {
    private $attributes = [];
    
    public function __get($name) {
        return $this->attributes[$name] ?? null;
    }
    
    public function __set($name, $value) {
        $this->attributes[$name] = $value;
    }
}

// Usage - no type safety
$product->price = '19.99'; // String accidentally
$product->stock = 'ten';   // String instead of int
$total = $product->price * $product->stock; // Runtime error
```

**After**:
```php
class Product {
    private array $attributes = [];
    
    public function getPrice(): ?float {
        return isset($this->attributes['price']) 
            ? (float) $this->attributes['price'] 
            : null;
    }
    
    public function setPrice(float $price): self {
        if ($price < 0) {
            throw new \InvalidArgumentException('Price must be positive');
        }
        $this->attributes['price'] = $price;
        return $this;
    }
    
    public function getStock(): int {
        return (int) ($this->attributes['stock'] ?? 0);
    }
    
    public function setStock(int $stock): self {
        if ($stock < 0) {
            throw new \InvalidArgumentException('Stock cannot be negative');
        }
        $this->attributes['stock'] = $stock;
        return $this;
    }
}

// Usage - compile-time type checking
$product->setPrice(19.99);  // ✓ Correct type
$product->setStock(10);     // ✓ Correct type
// $product->setPrice('19.99'); // ✗ Static analysis error
// $product->setStock('ten');   // ✗ Static analysis error
$total = $product->getPrice() * $product->getStock(); // Type-safe
```

### Pattern 3: Better Property Storage

**Before** (dynamic properties):
```php
class Config {
    public function __get($name) {
        return $this->$name ?? null; // Dynamic property access
    }
    
    public function __set($name, $value) {
        $this->$name = $value;
    }
}
```

**After** (explicit properties):
```php
class Config {
    private ?string $apiKey = null;
    private ?string $apiSecret = null;
    private int $timeout = 30;
    private bool $debug = false;
    
    public function getApiKey(): ?string {
        return $this->apiKey;
    }
    
    public function setApiKey(string $apiKey): self {
        $this->apiKey = $apiKey;
        return $this;
    }
    
    public function getApiSecret(): ?string {
        return $this->apiSecret;
    }
    
    public function setApiSecret(string $apiSecret): self {
        $this->apiSecret = $apiSecret;
        return $this;
    }
    
    public function getTimeout(): int {
        return $this->timeout;
    }
    
    public function setTimeout(int $timeout): self {
        if ($timeout <= 0) {
            throw new \InvalidArgumentException('Timeout must be positive');
        }
        $this->timeout = $timeout;
        return $this;
    }
    
    public function isDebug(): bool {
        return $this->debug;
    }
    
    public function setDebug(bool $debug): self {
        $this->debug = $debug;
        return $this;
    }
}
```

### Pattern 4: Migration with Deprecation

```php
class LegacyModel {
    private array $data = [];
    
    // New explicit methods
    public function getName(): ?string {
        return $this->data['name'] ?? null;
    }
    
    public function setName(string $name): self {
        $this->data['name'] = $name;
        return $this;
    }
    
    // Keep magic methods during transition
    /**
     * @deprecated Use getName() / setName() instead
     */
    public function __get(string $name) {
        $method = 'get' . ucfirst($name);
        if (method_exists($this, $method)) {
            trigger_error(
                sprintf('Direct property access is deprecated, use %s()', $method),
                E_USER_DEPRECATED
            );
            return $this->$method();
        }
        return $this->data[$name] ?? null;
    }
    
    /**
     * @deprecated Use explicit setters instead
     */
    public function __set(string $name, $value): void {
        $method = 'set' . ucfirst($name);
        if (method_exists($this, $method)) {
            trigger_error(
                sprintf('Direct property access is deprecated, use %s()', $method),
                E_USER_DEPRECATED
            );
            $this->$method($value);
            return;
        }
        $this->data[$name] = $value;
    }
}
```

## Type Inference Strategy

Infer types from usage patterns:

```php
// String property
$obj->name = "John";
$obj->email = "test@example.com";
→ public function getName(): ?string
→ public function setName(string $name): self

// Integer property
$obj->age = 25;
$obj->count = 10;
→ public function getAge(): int
→ public function setAge(int $age): self

// Boolean property
$obj->active = true;
$obj->enabled = false;
→ public function isActive(): bool
→ public function setActive(bool $active): self

// Float property
$obj->price = 19.99;
$obj->rate = 0.15;
→ public function getPrice(): float
→ public function setPrice(float $price): self

// Array property
$obj->tags = ['php', 'magento'];
$obj->items = [];
→ public function getTags(): array
→ public function setTags(array $tags): self

// Object property
$obj->config = new Config();
$obj->user = $userRepository->find(1);
→ public function getConfig(): ?Config
→ public function setConfig(Config $config): self
```

## Naming Conventions

### Getters
- `getName()` - for regular properties
- `isActive()` - for boolean properties (is/has/can)
- `hasItems()` - for existence checks
- `canExecute()` - for capability checks

### Setters
- `setName(string $name)` - standard setter
- `setActive(bool $active)` - boolean setter
- `enable()` / `disable()` - alternative to setActive(true/false)
- Return `self` for fluent interface

### Checks
- `hasProperty()` - replaces `isset($obj->property)`
- `isEmpty()` - for collection/container classes

### Removal
- `clearProperty()` - replaces `unset($obj->property)`
- `removeProperty()` - alternative naming

## Benefits

✅ **IDE Support**
- Autocomplete for all methods
- Jump to definition works
- Refactoring tools work correctly

✅ **Type Safety**
- Static analysis catches errors
- Return types documented
- Parameter validation at compile time

✅ **Maintainability**
- Explicit behavior (no hidden logic)
- Easier to understand for new developers
- Validation logic in one place

✅ **Documentation**
- PHPDoc describes each property
- Types visible in method signatures
- Self-documenting code

✅ **Debugging**
- Breakpoints in getters/setters work
- Stack traces show method names
- Easier to trace property changes

## Output Format

Generate a markdown report: `magic-methods-normalization-report.md`

```markdown
# Magic Methods Normalization Report

**Date:** YYYY-MM-DD
**Classes Analyzed:** N
**Properties to Convert:** N

## Summary

- Classes with magic methods: N
- Total properties: N
- Explicit methods to generate: N
- Consumers to update: N locations

## Class: User

**File:** src/Model/User.php
**Magic Methods:** __get, __set, __isset
**Properties Accessed:** name, email, age, active

### Generated Methods

```php
// Getters
public function getName(): ?string;
public function getEmail(): ?string;
public function getAge(): int;
public function isActive(): bool;

// Setters
public function setName(string $name): self;
public function setEmail(string $email): self;
public function setAge(int $age): self;
public function setActive(bool $active): self;

// Checks
public function hasName(): bool;
```

### Consumer Updates

**File:** src/Controller/UserController.php
**Line 45:** `$user->name` → `$user->getName()`
**Line 47:** `$user->email = $email` → `$user->setEmail($email)`

**File:** src/Service/UserService.php
**Line 23:** `isset($user->name)` → `$user->hasName()`

## Migration Strategy

### Phase 1: Generate Methods (Week 1)
- [ ] Generate all getter/setter methods
- [ ] Add type hints and validation
- [ ] Keep magic methods with deprecation warnings
- [ ] Update PHPDoc blocks

### Phase 2: Update Consumers (Week 2-3)
- [ ] Refactor all direct property access
- [ ] Update tests
- [ ] Verify with static analysis

### Phase 3: Remove Magic Methods (Week 4)
- [ ] Verify no magic method usage remains
- [ ] Remove `__get`, `__set`, `__isset`, `__unset`
- [ ] Final static analysis pass

## Static Analysis Configuration

```neon
# phpstan.neon
parameters:
    level: 8
    checkMissingIterableValueType: true
    checkGenericClassInNonGenericObjectType: true
```

## Next Steps

1. Review generated methods
2. Apply refactorings phase by phase
3. Update tests for each class
4. Run PHPStan/Psalm after each phase
5. Monitor deprecation warnings in logs
6. Remove magic methods when warnings stop
```

## Best Practices

1. **Fluent interfaces**: Return `$this` from setters for chaining
2. **Validation**: Add input validation in setters
3. **Type hints**: Use strict types (PHP 7.4+ union types, PHP 8.0+ mixed)
4. **Immutability**: Consider immutable objects with only getters
5. **Null safety**: Use nullable types where appropriate
6. **PHPDoc**: Document complex types (@return array<string, User>)

## Example Usage

**User request:**
```
This class uses magic methods and my IDE can't autocomplete. Can you refactor it?
```

**Claude execution:**
1. Analyzes class with magic methods
2. Scans codebase for property usage
3. Infers types from usage patterns
4. Generates explicit getter/setter methods
5. Shows migration plan with deprecation period
6. Updates consumers to use new methods
7. Runs static analysis to verify

## Quality Bar

- All properties have explicit typed methods
- No loss of functionality
- IDE autocomplete works for all properties
- Static analysis passes at level 8
- Tests updated and passing
- Migration path allows gradual rollout
