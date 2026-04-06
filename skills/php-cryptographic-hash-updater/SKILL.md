---
name: php-cryptographic-hash-updater
description: Audits codebase for obsolete hash functions (MD5, SHA1) and replaces with password_hash() using BCRYPT or ARGON2ID. Use when improving security, preparing for compliance audits, or modernizing authentication systems.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Cryptographic Hash Updater

Identifies and replaces insecure cryptographic hash functions (MD5, SHA1, crypt()) with modern, secure alternatives using `password_hash()` and `password_verify()` with BCRYPT or ARGON2ID algorithms.

## When to Use

- Security audit findings
- Compliance requirements (PCI-DSS, GDPR, etc.)
- Before penetration testing
- Modernizing legacy authentication
- After security vulnerability reports
- Upgrading from PHP < 5.5

## Security Context

### Insecure Hashing Methods
- **MD5**: Broken (collision attacks since 2004)
- **SHA1**: Deprecated (collision attacks since 2017)
- **Plain text**: Never acceptable
- **Simple hash + salt**: Vulnerable to rainbow tables
- **crypt()**: Manual salt management error-prone

### Modern Secure Methods
- **`password_hash()`**: Automatic salt generation
- **BCRYPT** (default): Industry standard, tuneable cost
- **ARGON2ID**: Winner of Password Hashing Competition, best for new projects
- **Key stretching**: Built-in resistance to brute force

## Execution Workflow

1. **Audit hash usage**
   - Grep for `md5()`, `sha1()`, `hash('sha1')`, `crypt()`
   - Find password storage patterns
   - Identify authentication logic
   - Locate hash comparisons

2. **Categorize findings**
   - **Critical**: Password hashing
   - **High**: API tokens, session IDs
   - **Medium**: Data integrity checks (may keep)
   - **Low**: Non-security checksums

3. **Analyze migration impact**
   - Check database schema (hash length)
   - Review authentication flows
   - Identify user registration logic
   - Plan migration for existing hashes

4. **Implement secure hashing**
   - Replace with `password_hash()` + `password_verify()`
   - Choose algorithm (BCRYPT or ARGON2ID)
   - Update database schema if needed
   - Implement migration for existing passwords

5. **Handle legacy data**
   - Implement re-hashing on login
   - Support old hash verification during transition
   - Force password reset if needed
   - Clean up old hashes after migration

6. **Verify security**
   - Test password verification
   - Verify hash strength
   - Check for timing attacks
   - Run security scan

## Refactoring Patterns

### Pattern 1: MD5 Password Hash

**Before** (INSECURE):
```php
// User registration
function registerUser(string $username, string $password): void {
    // INSECURE: MD5 is broken, no salt, rainbow table vulnerable
    $passwordHash = md5($password);
    
    $this->db->insert('users', [
        'username' => $username,
        'password' => $passwordHash, // 32 chars
    ]);
}

// User login
function authenticateUser(string $username, string $password): bool {
    $user = $this->db->fetchUser($username);
    
    // INSECURE: Vulnerable to timing attacks
    return $user && md5($password) === $user['password'];
}
```

**After** (SECURE):
```php
// User registration
function registerUser(string $username, string $password): void {
    // SECURE: Auto-salted, tunable cost, brute-force resistant
    $passwordHash = password_hash($password, PASSWORD_BCRYPT, [
        'cost' => 12, // Adjust based on performance testing
    ]);
    
    $this->db->insert('users', [
        'username' => $username,
        'password' => $passwordHash, // 60 chars for BCRYPT
    ]);
}

// User login
function authenticateUser(string $username, string $password): bool {
    $user = $this->db->fetchUser($username);
    
    if (!$user) {
        return false;
    }
    
    // SECURE: Timing-attack safe, automatic salt extraction
    return password_verify($password, $user['password']);
}
```

### Pattern 2: SHA1 with Manual Salt

**Before** (INSECURE):
```php
function hashPassword(string $password): array {
    // INSECURE: SHA1 is deprecated, manual salt management
    $salt = bin2hex(random_bytes(16));
    $hash = sha1($salt . $password);
    
    return [
        'hash' => $hash,
        'salt' => $salt,
    ];
}

function verifyPassword(string $password, string $hash, string $salt): bool {
    // INSECURE: Timing attack vulnerable
    return sha1($salt . $password) === $hash;
}
```

**After** (SECURE):
```php
function hashPassword(string $password): string {
    // SECURE: Automatic salt generation and management
    return password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
}

function verifyPassword(string $password, string $hash): bool {
    // SECURE: Constant-time comparison built-in
    return password_verify($password, $hash);
}
// No separate salt needed - embedded in hash
```

### Pattern 3: Migration Strategy (Gradual Re-hashing)

**Hybrid approach during transition:**
```php
class PasswordService {
    public function hashPassword(string $password): string {
        // Always use secure hashing for new passwords
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3,
        ]);
    }
    
    public function verifyAndUpgradePassword(
        string $password,
        string $storedHash,
        int $userId
    ): bool {
        // Check if it's a legacy hash (MD5 = 32 chars, no $)
        if (strlen($storedHash) === 32 && !str_starts_with($storedHash, '$')) {
            // Verify against legacy MD5 hash
            if (md5($password) === $storedHash) {
                // Password correct - upgrade to secure hash
                $newHash = $this->hashPassword($password);
                $this->updateUserPassword($userId, $newHash);
                return true;
            }
            return false;
        }
        
        // Modern hash - use password_verify
        $verified = password_verify($password, $storedHash);
        
        // Check if rehashing is needed (algorithm/cost changed)
        if ($verified && password_needs_rehash($storedHash, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3,
        ])) {
            // Upgrade hash parameters
            $newHash = $this->hashPassword($password);
            $this->updateUserPassword($userId, $newHash);
        }
        
        return $verified;
    }
}
```

### Pattern 4: Database Schema Migration

```php
// Database migration
class UpgradePasswordHashingMigration {
    public function up(): void {
        // Expand password column for BCRYPT (60 chars)
        $this->db->query("
            ALTER TABLE users 
            MODIFY COLUMN password VARCHAR(255) NOT NULL
        ");
        
        // Optional: Add column to track hash algorithm
        $this->db->query("
            ALTER TABLE users
            ADD COLUMN password_algorithm VARCHAR(20) DEFAULT 'bcrypt'
        ");
        
        // Optional: Force password reset for legacy hashes
        $this->db->query("
            UPDATE users 
            SET password_reset_required = 1
            WHERE LENGTH(password) = 32
        ");
    }
}
```

### Pattern 5: API Token Security

**Before** (INSECURE):
```php
function generateApiToken(int $userId): string {
    // INSECURE: Predictable, collision-prone
    return md5($userId . time());
}
```

**After** (SECURE):
```php
function generateApiToken(int $userId): string {
    // SECURE: Cryptographically secure random bytes
    $token = bin2hex(random_bytes(32)); // 64-char hex string
    
    // Store hashed version
    $this->db->insert('api_tokens', [
        'user_id' => $userId,
        'token_hash' => password_hash($token, PASSWORD_BCRYPT),
        'created_at' => time(),
    ]);
    
    // Return raw token to user (only shown once)
    return $token;
}

function verifyApiToken(string $token): ?int {
    $tokens = $this->db->query("SELECT * FROM api_tokens WHERE active = 1");
    
    foreach ($tokens as $row) {
        if (password_verify($token, $row['token_hash'])) {
            return $row['user_id'];
        }
    }
    
    return null;
}
```

## Algorithm Comparison

### BCRYPT (PASSWORD_BCRYPT)
```php
$hash = password_hash($password, PASSWORD_BCRYPT, [
    'cost' => 12, // 2^12 iterations (tunable)
]);

// Pros: Industry standard, widely supported, good security
// Cons: 72-byte password limit, limited cost scaling
// Output: 60 characters (includes algorithm, cost, salt, hash)
```

### ARGON2ID (PASSWORD_ARGON2ID - PHP 7.3+)
```php
$hash = password_hash($password, PASSWORD_ARGON2ID, [
    'memory_cost' => 65536, // 64 MB
    'time_cost' => 4,       // 4 iterations
    'threads' => 3,         // 3 parallel threads
]);

// Pros: Modern, resistant to GPU/ASIC attacks, tuneable
// Cons: Requires PHP 7.3+, more resource-intensive
// Output: ~96 characters
// Recommended for new projects
```

### DEFAULT (PASSWORD_DEFAULT)
```php
$hash = password_hash($password, PASSWORD_DEFAULT);

// Currently BCRYPT, may change in future PHP versions
// Automatic algorithm upgrades with PHP updates
```

## When to Keep Old Hash Functions

Acceptable uses for MD5/SHA1 (NOT for passwords):

### ✅ File Integrity Checks
```php
// OK: Non-security checksums for file deduplication
$fileHash = md5_file($path);
if (isset($seenFiles[$fileHash])) {
    // Duplicate file
}
```

### ✅ Cache Keys
```php
// OK: Non-cryptographic cache key generation
$cacheKey = 'user_' . md5($userId . $timestamp);
```

### ✅ Identifiers (Non-Security)
```php
// OK: Generate unique but non-security-critical IDs
$imageId = md5($imageData);
```

### ❌ NEVER for Security
- User passwords
- API tokens
- Session IDs
- Password reset tokens
- Authentication cookies

## Cost/Performance Tuning

Find optimal cost factor:
```php
function findOptimalCost(string $algorithm = PASSWORD_BCRYPT): int {
    $targetTime = 0.2; // 200ms target
    $cost = 8; // Starting cost
    
    do {
        $start = microtime(true);
        password_hash('benchmark', $algorithm, ['cost' => $cost]);
        $elapsed = microtime(true) - $start;
        
        if ($elapsed < $targetTime) {
            $cost++;
        }
    } while ($elapsed < $targetTime && $cost < 20);
    
    return $cost;
}

// Recommended: 10-12 for BCRYPT (100-250ms on modern hardware)
```

## Output Format

```markdown
# Cryptographic Hash Security Audit

**Date:** YYYY-MM-DD
**Files Scanned:** N
**Insecure Hash Usage:** N instances

## Critical Issues (Passwords)

### Issue 1: MD5 Password Hashing

**File:** src/Auth/UserService.php
**Line:** 45, 78
**Severity:** CRITICAL
**Risk:** Rainbow table attack, no salt, fast brute-force

**Current Code:**
```php
$passwordHash = md5($password);
```

**Secure Replacement:**
```php
$passwordHash = password_hash($password, PASSWORD_ARGON2ID, [
    'memory_cost' => 65536,
    'time_cost' => 4,
    'threads' => 3,
]);
```

**Migration Strategy:**
- Extend `users.password` column to VARCHAR(255)
- Implement gradual re-hashing on user login
- Estimated users to migrate: 12,450

## High Priority (Tokens)

### Issue 2: SHA1 API Token Generation

**File:** src/Api/TokenGenerator.php
**Line:** 23
**Severity:** HIGH
**Risk:** Predictable tokens, collision attacks

**Replacement:** Use `random_bytes(32)` + `password_hash()`

## Medium Priority (Non-Critical)

### Issue 3: MD5 for File Checksums

**File:** src/Storage/FileManager.php
**Line:** 67
**Severity:** MEDIUM
**Risk:** Low (file deduplication only)
**Action:** Consider upgrade to SHA256 or keep as-is

## Database Migrations Required

```sql
-- Expand password column
ALTER TABLE users MODIFY COLUMN password VARCHAR(255);

-- Add algorithm tracking
ALTER TABLE users ADD COLUMN password_algorithm VARCHAR(20) DEFAULT 'argon2id';
```

## Migration Timeline

- Week 1: Deploy new hashing code with hybrid verification
- Week 2-4: Users re-hash on login (monitor adoption)
- Week 5: Force password reset for remaining legacy hashes
- Week 6: Remove MD5/SHA1 verification code

## Security Improvements

- Brute-force resistance: 1000x stronger
- Rainbow table defense: Auto-salted
- Timing attack protection: Built-in
- Future-proof: Algorithm upgradeable
```

## Best Practices

1. **Use PASSWORD_ARGON2ID for new projects** (if PHP 7.3+)
2. **Use PASSWORD_DEFAULT for compatibility**
3. **Tune cost parameters** based on hardware
4. **Implement gradual re-hashing** for existing users
5. **Never store raw passwords** (even temporarily)
6. **Use password_needs_rehash()** to upgrade algorithms
7. **Test performance** under load before deploying

## Example Usage

**User request:**
```
Security audit found MD5 password hashing, need to fix ASAP
```

**Claude execution:**
1. Scans codebase for insecure hash functions
2. Identifies password hashing locations
3. Shows secure replacement code
4. Provides database migration script
5. Implements gradual re-hashing strategy
6. Verifies no insecure usage remains

## Quality Bar

- All password hashing uses `password_hash()`
- No MD5/SHA1 for security-critical operations
- Database schema supports modern hash lengths
- Migration strategy preserves user access
- Tests verify correct password verification
- Security scan confirms no vulnerabilities
