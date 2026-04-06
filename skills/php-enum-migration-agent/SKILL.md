---
name: php-enum-migration-agent
description: Converts scattered class constants used for state representation into native PHP 8.1 Enums. Use when modernizing PHP codebases, improving type safety, or refactoring state machines.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Enum Migration Agent

Identifies dispersed class constants representing states, types, or fixed value sets, and consolidates them into native PHP 8.1+ Enumerations (Enums). This improves type safety, enables exhaustive matching, and makes value sets explicit.

## When to Use

- Upgrading to PHP 8.1+
- Refactoring status/state constants
- Improving type safety in value objects
- Modernizing legacy constant-based enumerations
- When implementing type-safe state machines
- Before adding strict typing to a project

## Requirements

- PHP 8.1 or higher
- Projects using class constants for enumerations

## Execution Workflow

1. **Identify constant groups**
   - Grep for class constants with common prefixes
   - Find constants representing states/types/statuses
   - Group related constants by semantic meaning
   - Identify usage patterns across codebase

2. **Analyze constant usage**
   - Track comparisons: `$status === self::STATUS_PENDING`
   - Find switch statements over constants
   - Locate string/int constant values
   - Identify backed vs pure enum needs

3. **Design enum structure**
   - Choose Enum type (Pure or Backed)
   - Define case names and values
   - Plan additional methods (labels, colors, etc.)
   - Decide on enum location and namespace

4. **Generate enum classes**
   - Create enum with appropriate cases
   - Add helper methods (labels, from methods, etc.)
   - Implement value mapping if needed
   - Add PHPDoc for IDE support

5. **Refactor consumers**
   - Replace constant references with enum cases
   - Update type hints (string → Enum)
   - Refactor switch to match expressions
   - Update comparisons to use enum cases

6. **Remove old constants**
   - Mark constants as deprecated
   - Verify no usage remains
   - Remove constant definitions
   - Clean up imports

7. **Verify correctness**
   - Run tests
   - Check static analysis
   - Verify exhaustive matching works
   - Test serialization if applicable

## Enum Types

### Pure Enum (No Backing Value)
```php
enum Status {
    case Pending;
    case Approved;
    case Rejected;
}
```

### Backed Enum (String or Int)
```php
enum Status: string {
    case Pending = 'pending';
    case Approved = 'approved';
    case Rejected = 'rejected';
}

enum Priority: int {
    case Low = 1;
    case Medium = 2;
    case High = 3;
}
```

## Refactoring Patterns

### Pattern 1: Status Constants to String Enum

**Before**:
```php
class Order {
    public const STATUS_PENDING = 'pending';
    public const STATUS_PROCESSING = 'processing';
    public const STATUS_COMPLETED = 'completed';
    public const STATUS_CANCELLED = 'cancelled';
    
    private string $status;
    
    public function __construct() {
        $this->status = self::STATUS_PENDING;
    }
    
    public function getStatus(): string {
        return $this->status;
    }
    
    public function setStatus(string $status): void {
        // No type safety - any string accepted
        $this->status = $status;
    }
    
    public function isCompleted(): bool {
        return $this->status === self::STATUS_COMPLETED;
    }
}

// Usage
$order = new Order();
$order->setStatus(Order::STATUS_PROCESSING);
$order->setStatus('invalid_status'); // No error!
```

**After**:
```php
enum OrderStatus: string {
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
    
    public function label(): string {
        return match($this) {
            self::Pending => 'Pending',
            self::Processing => 'Processing',
            self::Completed => 'Completed',
            self::Cancelled => 'Cancelled',
        };
    }
    
    public function isTerminal(): bool {
        return match($this) {
            self::Completed, self::Cancelled => true,
            default => false,
        };
    }
}

class Order {
    private OrderStatus $status;
    
    public function __construct() {
        $this->status = OrderStatus::Pending;
    }
    
    public function getStatus(): OrderStatus {
        return $this->status;
    }
    
    public function setStatus(OrderStatus $status): void {
        // Type-safe - only OrderStatus enums accepted
        $this->status = $status;
    }
    
    public function isCompleted(): bool {
        return $this->status === OrderStatus::Completed;
    }
}

// Usage
$order = new Order();
$order->setStatus(OrderStatus::Processing); // ✓ Type-safe
// $order->setStatus('invalid'); // ✗ Compile error!
```

### Pattern 2: Integer Constants to Backed Int Enum

**Before**:
```php
class Logger {
    public const LEVEL_DEBUG = 1;
    public const LEVEL_INFO = 2;
    public const LEVEL_WARNING = 3;
    public const LEVEL_ERROR = 4;
    public const LEVEL_CRITICAL = 5;
    
    public function log(int $level, string $message): void {
        // Any int accepted, no validation
        if ($level >= self::LEVEL_ERROR) {
            $this->sendAlert($message);
        }
    }
}

$logger->log(Logger::LEVEL_ERROR, 'Error occurred');
$logger->log(999, 'Invalid level'); // No error
```

**After**:
```php
enum LogLevel: int {
    case Debug = 1;
    case Info = 2;
    case Warning = 3;
    case Error = 4;
    case Critical = 5;
    
    public function isAlertLevel(): bool {
        return $this->value >= self::Error->value;
    }
    
    public function color(): string {
        return match($this) {
            self::Debug => 'gray',
            self::Info => 'blue',
            self::Warning => 'yellow',
            self::Error => 'red',
            self::Critical => 'purple',
        };
    }
}

class Logger {
    public function log(LogLevel $level, string $message): void {
        // Type-safe - only LogLevel enums accepted
        if ($level->isAlertLevel()) {
            $this->sendAlert($message);
        }
    }
}

$logger->log(LogLevel::Error, 'Error occurred'); // ✓
// $logger->log(999, 'Invalid'); // ✗ Type error
```

### Pattern 3: Exhaustive Matching with Match Expression

**Before** (switch with no exhaustiveness check):
```php
class PaymentProcessor {
    public const METHOD_CREDIT_CARD = 'credit_card';
    public const METHOD_PAYPAL = 'paypal';
    public const METHOD_BANK_TRANSFER = 'bank_transfer';
    
    public function process(string $method, float $amount): void {
        switch ($method) {
            case self::METHOD_CREDIT_CARD:
                $this->processCreditCard($amount);
                break;
            case self::METHOD_PAYPAL:
                $this->processPaypal($amount);
                break;
            // Missing BANK_TRANSFER case - no compile error!
        }
    }
}
```

**After** (match with exhaustiveness check):
```php
enum PaymentMethod: string {
    case CreditCard = 'credit_card';
    case Paypal = 'paypal';
    case BankTransfer = 'bank_transfer';
}

class PaymentProcessor {
    public function process(PaymentMethod $method, float $amount): void {
        match ($method) {
            PaymentMethod::CreditCard => $this->processCreditCard($amount),
            PaymentMethod::Paypal => $this->processPaypal($amount),
            PaymentMethod::BankTransfer => $this->processBankTransfer($amount),
            // If a case is missing, PHPStan/IDE will error!
        };
    }
}
```

### Pattern 4: Enum with Methods and State Logic

**Before**:
```php
class Ticket {
    public const STATUS_OPEN = 'open';
    public const STATUS_IN_PROGRESS = 'in_progress';
    public const STATUS_RESOLVED = 'resolved';
    public const STATUS_CLOSED = 'closed';
    
    public function canTransitionTo(string $from, string $to): bool {
        $allowed = [
            self::STATUS_OPEN => [self::STATUS_IN_PROGRESS, self::STATUS_CLOSED],
            self::STATUS_IN_PROGRESS => [self::STATUS_RESOLVED, self::STATUS_OPEN],
            self::STATUS_RESOLVED => [self::STATUS_CLOSED, self::STATUS_OPEN],
            self::STATUS_CLOSED => [],
        ];
        
        return in_array($to, $allowed[$from] ?? []);
    }
}
```

**After**:
```php
enum TicketStatus: string {
    case Open = 'open';
    case InProgress = 'in_progress';
    case Resolved = 'resolved';
    case Closed = 'closed';
    
    public function allowedTransitions(): array {
        return match($this) {
            self::Open => [self::InProgress, self::Closed],
            self::InProgress => [self::Resolved, self::Open],
            self::Resolved => [self::Closed, self::Open],
            self::Closed => [],
        };
    }
    
    public function canTransitionTo(self $target): bool {
        return in_array($target, $this->allowedTransitions(), true);
    }
    
    public function isTerminal(): bool {
        return $this === self::Closed;
    }
}

// Usage
$status = TicketStatus::Open;
if ($status->canTransitionTo(TicketStatus::InProgress)) {
    // Transition allowed
}
```

## Common Enum Patterns

### Labels for Display
```php
enum UserRole: string {
    case Admin = 'admin';
    case Editor = 'editor';
    case Viewer = 'viewer';
    
    public function label(): string {
        return match($this) {
            self::Admin => 'Administrator',
            self::Editor => 'Content Editor',
            self::Viewer => 'Read-Only Viewer',
        };
    }
}
```

### Colors/Icons for UI
```php
enum TaskPriority: int {
    case Low = 1;
    case Medium = 2;
    case High = 3;
    case Urgent = 4;
    
    public function color(): string {
        return match($this) {
            self::Low => '#28a745',
            self::Medium => '#ffc107',
            self::High => '#fd7e14',
            self::Urgent => '#dc3545',
        };
    }
    
    public function icon(): string {
        return match($this) {
            self::Low => '▼',
            self::Medium => '■',
            self::High => '▲',
            self::Urgent => '⚠',
        };
    }
}
```

### Database/API Serialization
```php
enum OrderStatus: string {
    case Pending = 'pending';
    case Confirmed = 'confirmed';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    
    // From database string
    public static function fromString(string $value): self {
        return self::from($value); // Throws if invalid
    }
    
    public static function tryFromString(string $value): ?self {
        return self::tryFrom($value); // Returns null if invalid
    }
    
    // For JSON serialization
    public function jsonSerialize(): string {
        return $this->value;
    }
}
```

### Grouping Cases
```php
enum HttpStatus: int {
    case OK = 200;
    case Created = 201;
    case BadRequest = 400;
    case Unauthorized = 401;
    case NotFound = 404;
    case ServerError = 500;
    
    public function isSuccess(): bool {
        return $this->value >= 200 && $this->value < 300;
    }
    
    public function isClientError(): bool {
        return $this->value >= 400 && $this->value < 500;
    }
    
    public function isServerError(): bool {
        return $this->value >= 500;
    }
}
```

## Migration Strategy

### Phase 1: Create Enums (No Breaking Changes)
```php
// Keep old constants with deprecation
class Order {
    /** @deprecated Use OrderStatus enum */
    public const STATUS_PENDING = 'pending';
    
    // Add new enum property alongside old
    private OrderStatus $statusEnum;
    private string $status; // Keep temporarily
    
    public function __construct() {
        $this->statusEnum = OrderStatus::Pending;
        $this->status = $this->statusEnum->value; // Sync
    }
}
```

### Phase 2: Update Consumers
```php
// Update method signatures
public function setStatus(OrderStatus $status): void {
    $this->statusEnum = $status;
    $this->status = $status->value; // Keep in sync
}
```

### Phase 3: Remove Old Code
```php
// Remove old property and constants
class Order {
    private OrderStatus $status;
    
    public function setStatus(OrderStatus $status): void {
        $this->status = $status;
    }
}
```

## Output Format

```markdown
# Enum Migration Report

**Date:** YYYY-MM-DD
**Constant Groups Found:** N
**Enums to Create:** N

## Enum 1: OrderStatus

**Source:** src/Entity/Order.php
**Constants:** 4 (STATUS_*)
**Type:** Backed String Enum
**Usage Locations:** 23 files

### Generated Enum

```php
enum OrderStatus: string {
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
}
```

### Migration Tasks

- [ ] Create OrderStatus enum
- [ ] Update Order entity type hint
- [ ] Refactor 23 consumer files
- [ ] Remove old constants
- [ ] Update tests (12 files)

## Benefits Summary

- Type safety: Prevents invalid values
- Exhaustiveness: IDEs/PHPStan check all cases covered
- IDE support: Autocomplete for enum cases
- Encapsulation: Methods on enums (labels, validation)
- Performance: Same as constants (no overhead)
```

## Best Practices

1. **Use backed enums**: When storing/transmitting values (DB, API)
2. **Use pure enums**: For internal-only states
3. **Add helper methods**: labels(), color(), isValid(), etc.
4. **Leverage match**: For exhaustive case handling
5. **Serialize carefully**: Implement JsonSerializable if needed
6. **Test exhaustiveness**: Use PHPStan level 9
7. **Document transitions**: For state machine enums

## Quality Bar

- All constant groups converted to appropriate enum types
- Type hints updated throughout codebase
- Exhaustive matching verified with static analysis
- No behavioral changes
- Tests updated and passing
- IDE autocomplete working for all enum cases
