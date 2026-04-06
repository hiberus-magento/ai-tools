---
name: magento-plugin-interceptor-logic
description: Evaluates functional requirements and implements Magento 2 plugins (before, after, around interceptors) in di.xml without overloading execution flow. Use when extending core functionality, customizing business logic, or implementing integrations. Requires Adobe Commerce/Magento 2 knowledge.
allowed-tools: Read Grep Glob Bash Edit Write
model: opus
---

# Magento Plugin Interceptor Logic

Analyzes requirements and implements Magento 2 plugins (interceptors) using before, after, and around methods. Ensures minimal performance impact, follows best practices, and respects Magento architecture patterns.

## When to Use

- Extending core Magento functionality
- Modifying method behavior without rewriting classes
- Adding logging, validation, or side effects
- Intercepting service contracts
- Implementing integrations with external systems
- Customizing business logic
- Adding before/after hooks

## Magento Plugin Types

### Before Plugin
- Executes before the original method
- Can modify input parameters
- Cannot modify return value
- Use for: validation, logging, parameter modification

### After Plugin
- Executes after the original method
- Can modify return value
- Cannot modify parameters
- Use for: result transformation, additional processing, logging

### Around Plugin
- Wraps the original method completely
- Full control over execution
- Can prevent original method from running
- Most powerful but highest overhead
- Use sparingly: when full control needed

## Plugin Priority

```xml
<type name="Class\To\Intercept">
    <plugin name="vendor_module_plugin1" type="Vendor\Module\Plugin\Plugin1" sortOrder="10" />
    <plugin name="vendor_module_plugin2" type="Vendor\Module\Plugin\Plugin2" sortOrder="20" />
    <plugin name="vendor_module_plugin3" type="Vendor\Module\Plugin\Plugin3" sortOrder="30" disabled="false" />
</type>
```

- Lower sortOrder = earlier execution
- Default sortOrder = 0
- `disabled="true"` disables plugin

## Execution Workflow

1. **Analyze requirement**
   - Identify target class and method
   - Determine plugin type needed (before/after/around)
   - Check if class is interceptable
   - Review existing plugins

2. **Verify interceptability**
   - Must be public method
   - Must be in class (not trait/interface)
   - Must not be final class
   - Must not be final method
   - Must not be static method

3. **Choose plugin type**
   - **Before**: Need to modify input or validate
   - **After**: Need to modify output or log result
   - **Around**: Need full control or prevent execution

4. **Implement plugin class**
   - Follow naming convention: [Target]Plugin
   - Use dependency injection
   - Keep logic focused and minimal
   - Add proper PHPDoc

5. **Configure di.xml**
   - Define plugin in module's etc/di.xml (or frontend/adminhtml)
   - Use appropriate sortOrder
   - Use descriptive plugin name

6. **Test thoroughly**
   - Unit tests for plugin logic
   - Integration tests for interception
   - Performance testing
   - Test with other plugins active

## Implementation Patterns

### Pattern 1: Before Plugin (Parameter Modification)

**Requirement:** Add customer group validation before adding product to cart

```php
// File: app/code/Vendor/Module/Plugin/Quote/AddProductPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Quote;

use Magento\Quote\Model\Quote;
use Magento\Catalog\Model\Product;
use Magento\Framework\Exception\LocalizedException;
use Vendor\Module\Service\CustomerGroupValidator;

class AddProductPlugin
{
    private CustomerGroupValidator $groupValidator;
    
    public function __construct(
        CustomerGroupValidator $groupValidator
    ) {
        $this->groupValidator = $groupValidator;
    }
    
    /**
     * Validate customer group before adding product to cart
     *
     * @param Quote $subject
     * @param Product $product
     * @param float|null $qty
     * @return array Modified parameters
     * @throws LocalizedException
     */
    public function beforeAddProduct(
        Quote $subject,
        Product $product,
        $qty = null
    ): array {
        // Validation logic
        if (!$this->groupValidator->canPurchase($subject->getCustomer(), $product)) {
            throw new LocalizedException(
                __('Product is not available for your customer group.')
            );
        }
        
        // Return original parameters (or modified ones)
        return [$product, $qty];
    }
}
```

```xml
<!-- File: app/code/Vendor/Module/etc/frontend/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Quote\Model\Quote">
        <plugin name="vendor_module_add_product_validation" 
                type="Vendor\Module\Plugin\Quote\AddProductPlugin" 
                sortOrder="10" />
    </type>
</config>
```

### Pattern 2: After Plugin (Result Modification)

**Requirement:** Add custom data to product collection

```php
// File: app/code/Vendor/Module/Plugin/Catalog/ProductCollectionPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Catalog;

use Magento\Catalog\Model\ResourceModel\Product\Collection;
use Vendor\Module\Model\ResourceModel\ProductRating;

class ProductCollectionPlugin
{
    private ProductRating $productRating;
    
    public function __construct(ProductRating $productRating) {
        $this->productRating = $productRating;
    }
    
    /**
     * Add custom rating data after collection load
     *
     * @param Collection $subject
     * @param Collection $result
     * @return Collection
     */
    public function afterLoad(
        Collection $subject,
        Collection $result
    ): Collection {
        $productIds = $result->getAllIds();
        
        if (!empty($productIds)) {
            $ratings = $this->productRating->getRatingsByProductIds($productIds);
            
            foreach ($result as $product) {
                $productId = $product->getId();
                if (isset($ratings[$productId])) {
                    $product->setData('custom_rating', $ratings[$productId]);
                }
            }
        }
        
        return $result;
    }
}
```

```xml
<!-- File: app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Catalog\Model\ResourceModel\Product\Collection">
        <plugin name="vendor_module_add_product_rating" 
                type="Vendor\Module\Plugin\Catalog\ProductCollectionPlugin" 
                sortOrder="100" />
    </type>
</config>
```

### Pattern 3: Around Plugin (Full Control)

**Requirement:** Add caching layer to expensive API call

```php
// File: app/code/Vendor/Module/Plugin/Integration/ApiClientPlugin.php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Integration;

use Vendor\Integration\Model\ApiClient;
use Magento\Framework\App\CacheInterface;
use Closure;

class ApiClientPlugin
{
    private CacheInterface $cache;
    
    public function __construct(CacheInterface $cache) {
        $this->cache = $cache;
    }
    
    /**
     * Add caching layer around API call
     *
     * @param ApiClient $subject
     * @param Closure $proceed
     * @param string $endpoint
     * @param array $params
     * @return array
     */
    public function aroundFetchData(
        ApiClient $subject,
        Closure $proceed,
        string $endpoint,
        array $params = []
    ): array {
        $cacheKey = 'api_' . md5($endpoint . json_encode($params));
        
        // Try cache first
        $cached = $this->cache->load($cacheKey);
        if ($cached !== false) {
            return json_decode($cached, true);
        }
        
        // Call original method
        $result = $proceed($endpoint, $params);
        
        // Cache result
        $this->cache->save(
            json_encode($result),
            $cacheKey,
            ['api_cache'],
            3600
        );
        
        return $result;
    }
}
```

```xml
<!-- File: app/code/Vendor/Module/etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Vendor\Integration\Model\ApiClient">
        <plugin name="vendor_module_api_cache" 
                type="Vendor\Module\Plugin\Integration\ApiClientPlugin" 
                sortOrder="10" />
    </type>
</config>
```

### Pattern 4: Multiple Plugins on Same Method

```php
// Plugin 1: Validation (sortOrder="10")
class ValidationPlugin
{
    public function beforeSave(
        \Magento\Catalog\Model\Product $subject
    ): array {
        $this->validate($subject);
        return [];
    }
}

// Plugin 2: Logging (sortOrder="20")
class LoggingPlugin
{
    public function afterSave(
        \Magento\Catalog\Model\Product $subject,
        $result
    ) {
        $this->logger->info('Product saved: ' . $subject->getSku());
        return $result;
    }
}

// Plugin 3: Cache invalidation (sortOrder="30")
class CachePlugin
{
    public function afterSave(
        \Magento\Catalog\Model\Product $subject,
        $result
    ) {
        $this->cacheManager->clean(['product_' . $subject->getId()]);
        return $result;
    }
}
```

## Best Practices

### ✅ DO

1. **Prefer before/after over around**
   - Around plugins have performance overhead
   - Before/after are more maintainable
   - Easier to debug plugin chains

2. **Keep plugins focused**
   - Single responsibility
   - Minimal logic
   - Extract complex logic to services

3. **Use dependency injection**
   - Inject services via constructor
   - Don't use ObjectManager directly

4. **Return correct types**
   - Before: array of parameters
   - After: modified return value
   - Around: return $proceed() result

5. **Handle plugin chains**
   - Around: always call $proceed()
   - Before: return all parameters (even unmodified)
   - After: always return a value

6. **Use appropriate scope**
   - `etc/di.xml`: Global
   - `etc/frontend/di.xml`: Storefront only
   - `etc/adminhtml/di.xml`: Admin only

### ❌ DON'T

1. **Don't intercept non-public methods**
2. **Don't use plugins on final classes**
3. **Don't create circular dependencies**
4. **Don't forget to return values**
5. **Don't put business logic in plugins** (use services)
6. **Don't nest around plugins** (causes exponential overhead)
7. **Don't use plugins for every customization** (preferences, observers, events may be better)

## Plugin vs Other Patterns

| Use Case | Pattern | Reason |
|----------|---------|--------|
| Modify method behavior | Plugin | Non-intrusive, composable |
| Replace entire class | Preference | When rewrite is necessary |
| React to events | Observer | Decoupled, multiple listeners |
| Add new functionality | Service class | Clean separation |
| Modify data before save | Plugin | Intercept save method |
| Respond to entity save | Observer | Entity-specific events |

## Common Plugin Targets

### Catalog
- `Magento\Catalog\Model\Product::save()`
- `Magento\Catalog\Model\Product::load()`
- `Magento\Catalog\Model\ResourceModel\Product\Collection::load()`

### Quote/Cart
- `Magento\Quote\Model\Quote::addProduct()`
- `Magento\Quote\Api\CartRepositoryInterface::save()`
- `Magento\Quote\Model\Quote\Item\ToOrderItem::convert()`

### Order
- `Magento\Sales\Api\OrderRepositoryInterface::save()`
- `Magento\Sales\Model\Order::place()`
- `Magento\Sales\Model\Order\Email\Sender\OrderSender::send()`

### Customer
- `Magento\Customer\Api\AccountManagementInterface::createAccount()`
- `Magento\Customer\Api\CustomerRepositoryInterface::save()`

### Checkout
- `Magento\Checkout\Model\ShippingInformationManagement::saveAddressInformation()`
- `Magento\Checkout\Api\PaymentInformationManagementInterface::savePaymentInformationAndPlaceOrder()`

## Testing Plugins

### Unit Test
```php
<?php
namespace Vendor\Module\Test\Unit\Plugin;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Plugin\Quote\AddProductPlugin;

class AddProductPluginTest extends TestCase
{
    private $plugin;
    private $groupValidator;
    
    protected function setUp(): void
    {
        $this->groupValidator = $this->createMock(\Vendor\Module\Service\CustomerGroupValidator::class);
        $this->plugin = new AddProductPlugin($this->groupValidator);
    }
    
    public function testBeforeAddProductWithValidGroup(): void
    {
        $quote = $this->createMock(\Magento\Quote\Model\Quote::class);
        $product = $this->createMock(\Magento\Catalog\Model\Product::class);
        
        $this->groupValidator->expects($this->once())
            ->method('canPurchase')
            ->willReturn(true);
        
        $result = $this->plugin->beforeAddProduct($quote, $product, 1);
        
        $this->assertEquals([$product, 1], $result);
    }
    
    public function testBeforeAddProductWithInvalidGroup(): void
    {
        $this->expectException(\Magento\Framework\Exception\LocalizedException::class);
        
        $quote = $this->createMock(\Magento\Quote\Model\Quote::class);
        $product = $this->createMock(\Magento\Catalog\Model\Product::class);
        
        $this->groupValidator->expects($this->once())
            ->method('canPurchase')
            ->willReturn(false);
        
        $this->plugin->beforeAddProduct($quote, $product, 1);
    }
}
```

### Integration Test
```php
<?php
namespace Vendor\Module\Test\Integration\Plugin;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class AddProductPluginTest extends TestCase
{
    private $quote;
    private $product;
    
    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->quote = $objectManager->create(\Magento\Quote\Model\Quote::class);
        $this->product = $objectManager->create(\Magento\Catalog\Model\Product::class);
    }
    
    /**
     * @magentoAppIsolation enabled
     * @magentoDataFixture loadProductFixture
     */
    public function testPluginInterceptsAddProduct(): void
    {
        $this->product->load(1);
        
        try {
            $this->quote->addProduct($this->product, 1);
            // Plugin should have intercepted
        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            $this->assertStringContains('not available', $e->getMessage());
        }
    }
}
```

## Debugging Plugins

### Enable plugin logging
```php
// In plugin method
$this->logger->debug('Plugin executed', [
    'subject_class' => get_class($subject),
    'parameters' => $params,
]);
```

### Check plugin registration
```bash
bin/magento dev:di:info "Magento\Catalog\Model\Product"
```

### List all plugins for a class
```bash
bin/magento dev:di:info "Magento\Quote\Model\Quote" --plugins
```

## Performance Considerations

1. **Avoid around plugins when possible** (2x overhead vs before/after)
2. **Use sortOrder wisely** (earlier = less overhead if throwing exceptions)
3. **Cache plugin results** when applicable
4. **Don't load heavy dependencies** in constructor
5. **Profile with Blackfire/New Relic** to measure impact

## Output Format

```markdown
# Plugin Implementation Plan

**Requirement:** [Description]
**Target Class:** Magento\Catalog\Model\Product
**Target Method:** save()
**Plugin Type:** Before
**Reason:** Need to validate data before save

## Implementation

### Plugin Class
**File:** app/code/Vendor/Module/Plugin/Catalog/ProductValidationPlugin.php

```php
[Generated plugin code]
```

### DI Configuration
**File:** app/code/Vendor/Module/etc/di.xml

```xml
[Generated di.xml configuration]
```

### Tests
**File:** app/code/Vendor/Module/Test/Unit/Plugin/Catalog/ProductValidationPluginTest.php

## Performance Impact

- Estimated overhead: < 1ms per call
- sortOrder: 10 (early execution)
- Scope: Global

## Deployment Steps

1. Deploy plugin code
2. Run `bin/magento setup:di:compile`
3. Run `bin/magento cache:flush`
4. Test in staging
5. Monitor performance in production
```

## Quality Bar

- Plugin intercepts correct class and method
- Correct plugin type chosen (before/after/around)
- Follows Magento coding standards
- Proper dependency injection
- Unit and integration tests included
- Performance impact minimal
- Properly documented with PHPDoc
