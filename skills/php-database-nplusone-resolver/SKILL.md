---
name: php-database-nplusone-resolver
description: Identifies N+1 query problems in loops and refactors to use eager loading or batch collection. Use when facing performance issues, slow API responses, or excessive database queries.
allowed-tools: Read Grep Glob Bash Edit
model: sonnet
---

# PHP Database N+1 Query Resolver

Detects and resolves N+1 query problems where iterative database queries inside loops cause performance degradation. Refactors code to use eager loading, batch fetching, or query optimization techniques.

## When to Use

- Slow page loads or API responses
- High database query counts in logs
- Performance profiling shows many repeated queries
- ORM-based code with relationship loading
- Before production deployment of data-intensive features
- When database connection limits are reached

## Problem Pattern

**N+1 Query Problem:**
```php
// 1 query to get all users
$users = $userRepository->findAll(); // Query 1

foreach ($users as $user) {
    // N queries (one per user)
    echo $user->getAddress()->getCity(); // Query 2, 3, 4, ... N+1
}
// Total: 1 + N queries
```

## Execution Workflow

1. **Identify N+1 patterns**
   - Grep for database queries inside loops
   - Find ORM relationship access in iterations
   - Locate repository calls within foreach
   - Check for lazy loading patterns

2. **Analyze query impact**
   - Count queries per iteration
   - Estimate data volumes
   - Profile with SQL query logging
   - Calculate performance impact

3. **Choose optimization strategy**
   - **Eager loading**: Preload relationships
   - **Batch fetching**: Load IDs, fetch in batches
   - **Query optimization**: Use JOINs
   - **Caching**: Cache repeated queries

4. **Implement solution**
   - Apply eager loading directives
   - Refactor to batch collection
   - Add query result caching
   - Optimize database schema if needed

5. **Verify improvement**
   - Count queries before/after
   - Measure response time improvement
   - Profile with production-like data
   - Monitor query execution plans

## Refactoring Patterns

### Pattern 1: ORM Eager Loading (Doctrine/Eloquent)

**Before** (N+1 problem):
```php
// 1 query
$users = $entityManager->getRepository(User::class)->findAll();

foreach ($users as $user) {
    // N queries (lazy loading)
    echo $user->getAddress()->getStreet();
    echo $user->getOrders()->count();
}
// Total: 1 + N + N = 2N + 1 queries
```

**After** (eager loading):
```php
// 1-3 queries total (depending on JOIN strategy)
$users = $entityManager->getRepository(User::class)
    ->createQueryBuilder('u')
    ->leftJoin('u.address', 'a')
    ->addSelect('a')
    ->leftJoin('u.orders', 'o')
    ->addSelect('o')
    ->getQuery()
    ->getResult();

foreach ($users as $user) {
    // No additional queries
    echo $user->getAddress()->getStreet();
    echo $user->getOrders()->count();
}
// Total: 3 queries (fixed, regardless of N)
```

### Pattern 2: Batch Fetching with IN Clause

**Before**:
```php
$products = $productRepository->findAll(); // 1 query

foreach ($products as $product) {
    // N queries
    $category = $categoryRepository->find($product->getCategoryId());
    echo $category->getName();
}
```

**After**:
```php
$products = $productRepository->findAll(); // 1 query

// Extract all category IDs
$categoryIds = array_map(
    fn($p) => $p->getCategoryId(),
    $products
);

// Batch fetch all categories (1 query)
$categories = $categoryRepository->findBy(['id' => $categoryIds]);
$categoriesById = [];
foreach ($categories as $category) {
    $categoriesById[$category->getId()] = $category;
}

// Use pre-fetched data
foreach ($products as $product) {
    $category = $categoriesById[$product->getCategoryId()];
    echo $category->getName();
}
// Total: 2 queries (fixed)
```

### Pattern 3: Query Result Caching

**Before**:
```php
foreach ($orders as $order) {
    // Repeated query for same customer
    $customer = $customerRepository->find($order->getCustomerId());
    echo $customer->getName();
}
```

**After**:
```php
$customerCache = [];

foreach ($orders as $order) {
    $customerId = $order->getCustomerId();
    
    if (!isset($customerCache[$customerId])) {
        $customerCache[$customerId] = $customerRepository->find($customerId);
    }
    
    $customer = $customerCache[$customerId];
    echo $customer->getName();
}
```

### Pattern 4: Single Optimized Query with JOINs

**Before**:
```php
$orders = $db->query("SELECT * FROM orders"); // 1 query

foreach ($orders as $order) {
    // N queries
    $customer = $db->query("SELECT * FROM customers WHERE id = ?", [$order['customer_id']]);
    $items = $db->query("SELECT * FROM order_items WHERE order_id = ?", [$order['id']]);
    
    processOrder($order, $customer, $items);
}
```

**After**:
```php
// Single query with JOINs
$query = "
    SELECT 
        o.*,
        c.name as customer_name,
        c.email as customer_email,
        GROUP_CONCAT(oi.product_id) as product_ids,
        GROUP_CONCAT(oi.quantity) as quantities
    FROM orders o
    INNER JOIN customers c ON o.customer_id = c.id
    LEFT JOIN order_items oi ON o.id = oi.order_id
    GROUP BY o.id
";

$orders = $db->query($query);

foreach ($orders as $order) {
    processOrder($order); // All data already loaded
}
```

## Detection Patterns

Look for these anti-patterns:

### Foreach + Query
```php
foreach ($collection as $item) {
    $related = $repository->find($item->getRelatedId()); // ⚠️ N+1
}
```

### Foreach + ORM Lazy Load
```php
foreach ($users as $user) {
    $address = $user->getAddress(); // ⚠️ Lazy load = N queries
}
```

### Foreach + Database Call
```php
foreach ($ids as $id) {
    $result = $db->query("SELECT * FROM table WHERE id = ?", [$id]); // ⚠️ N+1
}
```

### Nested Foreach with Queries
```php
foreach ($categories as $category) {
    foreach ($category->getProducts() as $product) { // ⚠️ N * M queries
        echo $product->getManufacturer()->getName(); // ⚠️ Additional N * M queries
    }
}
```

## ORM-Specific Solutions

### Doctrine
```php
// Eager loading with QueryBuilder
$users = $repository->createQueryBuilder('u')
    ->leftJoin('u.address', 'a')
    ->addSelect('a')
    ->getQuery()
    ->getResult();

// DQL with FETCH JOIN
$dql = "SELECT u, a FROM User u JOIN u.address a";
$users = $entityManager->createQuery($dql)->getResult();
```

### Eloquent (Laravel)
```php
// Eager loading
$users = User::with('address', 'orders')->get();

// Lazy eager loading
$users = User::all();
$users->load('address', 'orders');

// Nested eager loading
$users = User::with('orders.items.product')->get();
```

### Custom ORM/Raw SQL
```php
// Use IN clause for batch fetching
$ids = [1, 2, 3, 4, 5];
$query = "SELECT * FROM table WHERE id IN (" . implode(',', $ids) . ")";

// Use JOINs for related data
$query = "
    SELECT u.*, a.street, a.city
    FROM users u
    LEFT JOIN addresses a ON u.id = a.user_id
";
```

## Performance Metrics

Track these metrics:

- **Query count**: Before vs After
- **Response time**: Page load or API response
- **Database CPU**: Query execution time
- **Memory usage**: Loading strategy impact

## Output Format

```markdown
# N+1 Query Resolution Report

**Date:** YYYY-MM-DD
**Files Analyzed:** N
**N+1 Problems Found:** N

## Critical Issues

### Issue 1: User Address Loading

**File:** src/Controller/UserController.php
**Line:** 45-52
**Severity:** CRITICAL
**Impact:** 500 users = 501 queries (0.5s → 5.2s page load)

**Before:**
```php
$users = $userRepository->findAll(); // 1 query
foreach ($users as $user) {
    echo $user->getAddress()->getCity(); // 500 queries
}
```

**After:**
```php
$users = $userRepository->createQueryBuilder('u')
    ->leftJoin('u.address', 'a')
    ->addSelect('a')
    ->getQuery()
    ->getResult(); // 1 query

foreach ($users as $user) {
    echo $user->getAddress()->getCity(); // 0 additional queries
}
```

**Performance Gain:**
- Queries: 501 → 1 (99.8% reduction)
- Response time: 5.2s → 0.5s (90% faster)

## Performance Summary

| Endpoint | Before Queries | After Queries | Improvement |
|----------|---------------|---------------|-------------|
| /users | 501 | 1 | 99.8% |
| /orders | 1,253 | 3 | 99.7% |
| /products | 867 | 2 | 99.7% |

## Next Steps

1. Apply eager loading refactorings
2. Enable query logging in dev environment
3. Profile with production data volumes
4. Monitor database metrics post-deployment
```

## Testing

```php
// Test query count with query logger
public function testNoNPlusOneQueries(): void {
    $queryLogger = new QueryLogger();
    $this->entityManager->getConnection()
        ->getConfiguration()
        ->setSQLLogger($queryLogger);
    
    $users = $this->userRepository->findAllWithAddresses();
    
    // Should be 1 query, not N+1
    $this->assertCount(1, $queryLogger->queries);
    
    // Verify data is loaded
    foreach ($users as $user) {
        $this->assertNotNull($user->getAddress());
    }
    
    // Still just 1 query (no lazy loads)
    $this->assertCount(1, $queryLogger->queries);
}
```

## Best Practices

1. **Profile first**: Use query logging to confirm N+1 exists
2. **Eager load relationships**: Preload what you'll use
3. **Batch fetch when needed**: Use IN clauses for flexible loading
4. **Monitor query counts**: Track in production metrics
5. **Test with realistic data**: 10 records won't show the problem
6. **Use database indexes**: Ensure JOIN columns are indexed
7. **Consider caching**: For frequently accessed data

## Quality Bar

- All N+1 patterns identified with query counts
- Refactorings maintain exact behavior
- Query count verified with profiling
- Performance improvement measured
- No breaking changes to public APIs
- Tests verify no additional queries fired
