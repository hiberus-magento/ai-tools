---
name: magento-graphql-endpoint-resolver
description: Generates schema.graphqls files and implements PHP resolvers to enable GraphQL data fetching for PWA architectures. Use when building headless storefronts, mobile apps, or custom admin interfaces. Requires Adobe Commerce/Magento 2 GraphQL knowledge.
allowed-tools: Read Grep Glob Bash Edit Write
model: opus
---

# Magento GraphQL Endpoint Resolver

Creates GraphQL schema definitions and implements corresponding PHP resolvers for headless Adobe Commerce/Magento 2 architectures (PWA Studio, custom frontends, mobile apps).

## When to Use

- Building PWA storefronts
- Creating headless commerce experiences
- Developing mobile apps
- Custom admin interfaces
- Third-party integrations via GraphQL
- Exposing custom data to frontend
- Extending Magento GraphQL API

## Requirements

- Magento 2.3.0+ (GraphQL support)
- Understanding of GraphQL schema language
- Knowledge of Magento service contracts

## Execution Workflow

1. **Analyze data requirements**
   - Identify entities to expose
   - Define query/mutation needs
   - Map to existing service contracts
   - Plan resolver implementation

2. **Design GraphQL schema**
   - Define types, queries, mutations
   - Plan input/output types
   - Consider pagination, filtering
   - Follow GraphQL best practices

3. **Create schema.graphqls**
   - Define schema in etc/schema.graphqls
   - Use Magento conventions
   - Add documentation annotations

4. **Implement resolvers**
   - Create resolver classes
   - Implement ResolverInterface
   - Use service contracts
   - Handle errors properly

5. **Configure dependency injection**
   - Map schema to resolvers in di.xml
   - Inject required services

6. **Test queries/mutations**
   - Use GraphQL IDE (GraphiQL, Altair)
   - Write integration tests
   - Test authorization

## GraphQL Schema Components

### Type Definition
```graphql
type CustomProduct {
    sku: String! @doc(description: "Product SKU")
    name: String @doc(description: "Product name")
    custom_rating: Float @doc(description: "Custom rating score")
    is_available: Boolean!
}
```

### Query Definition
```graphql
type Query {
    customProducts(
        filter: ProductFilterInput @doc(description: "Filter criteria")
        pageSize: Int = 20 @doc(description: "Items per page")
        currentPage: Int = 1 @doc(description: "Current page number")
    ): CustomProductsOutput @resolver(class: "Vendor\\Module\\Model\\Resolver\\Products") @doc(description: "Fetch custom products")
}
```

### Mutation Definition
```graphql
type Mutation {
    addProductRating(
        input: AddProductRatingInput! @doc(description: "Rating input data")
    ): AddProductRatingOutput @resolver(class: "Vendor\\Module\\Model\\Resolver\\AddProductRating") @doc(description: "Add product rating")
}
```

### Input Type
```graphql
input AddProductRatingInput {
    sku: String! @doc(description: "Product SKU")
    rating: Int! @doc(description: "Rating value 1-5")
    comment: String @doc(description: "Optional comment")
}
```

### Output Type
```graphql
type AddProductRatingOutput {
    success: Boolean! @doc(description: "Operation success status")
    message: String @doc(description: "Result message")
    rating: ProductRating @doc(description: "Created rating")
}
```

## Implementation Patterns

### Pattern 1: Simple Query Resolver

**File:** `app/code/Vendor/Module/etc/schema.graphqls`
```graphql
type Query {
    customProductBySku(sku: String!): CustomProduct 
        @resolver(class: "Vendor\\Module\\Model\\Resolver\\ProductBySku")
        @doc(description: "Get product by SKU with custom data")
}

type CustomProduct {
    sku: String!
    name: String
    custom_rating: Float
}
```

**File:** `app/code/Vendor/Module/Model/Resolver/ProductBySku.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Exception\GraphQlNoSuchEntityException;
use Vendor\Module\Api\RatingRepositoryInterface;

class ProductBySku implements ResolverInterface
{
    private ProductRepositoryInterface $productRepository;
    private RatingRepositoryInterface $ratingRepository;
    
    public function __construct(
        ProductRepositoryInterface $productRepository,
        RatingRepositoryInterface $ratingRepository
    ) {
        $this->productRepository = $productRepository;
        $this->ratingRepository = $ratingRepository;
    }
    
    /**
     * @inheritdoc
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        if (empty($args['sku'])) {
            throw new GraphQlInputException(__('SKU is required'));
        }
        
        try {
            $product = $this->productRepository->get($args['sku']);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            throw new GraphQlNoSuchEntityException(__('Product not found'));
        }
        
        $rating = $this->ratingRepository->getRatingBySku($args['sku']);
        
        return [
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'custom_rating' => $rating ? $rating->getScore() : null,
        ];
    }
}
```

### Pattern 2: Collection Query with Pagination

**Schema:**
```graphql
type Query {
    customProducts(
        filter: ProductFilterInput
        pageSize: Int = 20
        currentPage: Int = 1
    ): CustomProductsOutput
        @resolver(class: "Vendor\\Module\\Model\\Resolver\\Products")
}

input ProductFilterInput {
    sku: FilterTypeInput
    name: FilterTypeInput
    min_rating: Float
}

type CustomProductsOutput {
    items: [CustomProduct!]!
    page_info: SearchResultPageInfo!
    total_count: Int!
}
```

**Resolver:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Vendor\Module\Model\ResourceModel\Product\CollectionFactory;
use Magento\Framework\Api\SearchCriteriaBuilder;

class Products implements ResolverInterface
{
    private CollectionFactory $collectionFactory;
    private SearchCriteriaBuilder $searchCriteriaBuilder;
    
    public function __construct(
        CollectionFactory $collectionFactory,
        SearchCriteriaBuilder $searchCriteriaBuilder
    ) {
        $this->collectionFactory = $collectionFactory;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    }
    
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $collection = $this->collectionFactory->create();
        
        // Apply filters
        if (isset($args['filter'])) {
            $this->applyFilters($collection, $args['filter']);
        }
        
        // Pagination
        $pageSize = $args['pageSize'] ?? 20;
        $currentPage = $args['currentPage'] ?? 1;
        $collection->setPageSize($pageSize);
        $collection->setCurPage($currentPage);
        
        $items = [];
        foreach ($collection as $item) {
            $items[] = [
                'sku' => $item->getSku(),
                'name' => $item->getName(),
                'custom_rating' => $item->getData('custom_rating'),
            ];
        }
        
        return [
            'items' => $items,
            'page_info' => [
                'page_size' => $pageSize,
                'current_page' => $currentPage,
                'total_pages' => $collection->getLastPageNumber(),
            ],
            'total_count' => $collection->getSize(),
        ];
    }
    
    private function applyFilters($collection, array $filters): void
    {
        if (isset($filters['sku'])) {
            $collection->addFieldToFilter('sku', $filters['sku']);
        }
        if (isset($filters['name'])) {
            $collection->addFieldToFilter('name', ['like' => '%' . $filters['name'] . '%']);
        }
        if (isset($filters['min_rating'])) {
            $collection->addFieldToFilter('custom_rating', ['gteq' => $filters['min_rating']]);
        }
    }
}
```

### Pattern 3: Mutation Resolver

**Schema:**
```graphql
type Mutation {
    addProductRating(input: AddProductRatingInput!): AddProductRatingOutput
        @resolver(class: "Vendor\\Module\\Model\\Resolver\\AddProductRating")
}

input AddProductRatingInput {
    sku: String!
    rating: Int!
    comment: String
}

type AddProductRatingOutput {
    success: Boolean!
    message: String
    rating: ProductRating
}

type ProductRating {
    id: Int!
    sku: String!
    rating: Int!
    comment: String
    created_at: String
}
```

**Resolver:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;
use Vendor\Module\Api\RatingRepositoryInterface;
use Vendor\Module\Api\Data\RatingInterfaceFactory;

class AddProductRating implements ResolverInterface
{
    private RatingRepositoryInterface $ratingRepository;
    private RatingInterfaceFactory $ratingFactory;
    
    public function __construct(
        RatingRepositoryInterface $ratingRepository,
        RatingInterfaceFactory $ratingFactory
    ) {
        $this->ratingRepository = $ratingRepository;
        $this->ratingFactory = $ratingFactory;
    }
    
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        // Check customer authorization
        if (!$context->getUserId()) {
            throw new GraphQlAuthorizationException(__('Customer must be logged in'));
        }
        
        // Validate input
        $this->validateInput($args['input']);
        
        // Create rating
        $rating = $this->ratingFactory->create();
        $rating->setSku($args['input']['sku']);
        $rating->setRating($args['input']['rating']);
        $rating->setComment($args['input']['comment'] ?? '');
        $rating->setCustomerId($context->getUserId());
        
        try {
            $savedRating = $this->ratingRepository->save($rating);
            
            return [
                'success' => true,
                'message' => __('Rating added successfully'),
                'rating' => [
                    'id' => $savedRating->getId(),
                    'sku' => $savedRating->getSku(),
                    'rating' => $savedRating->getRating(),
                    'comment' => $savedRating->getComment(),
                    'created_at' => $savedRating->getCreatedAt(),
                ],
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'message' => __('Failed to add rating: %1', $e->getMessage()),
                'rating' => null,
            ];
        }
    }
    
    private function validateInput(array $input): void
    {
        if (empty($input['sku'])) {
            throw new GraphQlInputException(__('SKU is required'));
        }
        
        if (!isset($input['rating']) || $input['rating'] < 1 || $input['rating'] > 5) {
            throw new GraphQlInputException(__('Rating must be between 1 and 5'));
        }
    }
}
```

### Pattern 4: Extending Existing Types

**Schema:**
```graphql
# Extend Magento's built-in ProductInterface
type ProductInterface {
    custom_rating: Float @resolver(class: "Vendor\\Module\\Model\\Resolver\\Product\\CustomRating")
    @doc(description: "Custom product rating")
}
```

**Field Resolver:**
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Resolver\Product;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Vendor\Module\Api\RatingRepositoryInterface;

class CustomRating implements ResolverInterface
{
    private RatingRepositoryInterface $ratingRepository;
    
    public function __construct(RatingRepositoryInterface $ratingRepository) {
        $this->ratingRepository = $ratingRepository;
    }
    
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        if (!isset($value['sku'])) {
            return null;
        }
        
        $rating = $this->ratingRepository->getRatingBySku($value['sku']);
        
        return $rating ? $rating->getScore() : null;
    }
}
```

## Common GraphQL Patterns

### Authorization Check
```php
// Check if customer is logged in
if (!$context->getUserId()) {
    throw new GraphQlAuthorizationException(__('Customer must be logged in'));
}

// Check customer ID matches
if ($context->getUserId() != $requestedCustomerId) {
    throw new GraphQlAuthorizationException(__('Unauthorized access'));
}
```

### Error Handling
```php
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Exception\GraphQlNoSuchEntityException;
use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;

try {
    // Operation
} catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
    throw new GraphQlNoSuchEntityException(__($e->getMessage()));
} catch (\Exception $e) {
    throw new GraphQlInputException(__('An error occurred: %1', $e->getMessage()));
}
```

### Context Usage
```php
// Get customer ID
$customerId = $context->getUserId();

// Get store ID
$storeId = $context->getExtensionAttributes()->getStore()->getId();

// Check if customer is guest
$isGuest = $context->getUserType() === UserContextInterface::USER_TYPE_GUEST;
```

## Testing GraphQL Endpoints

### Query Example
```graphql
query {
  customProductBySku(sku: "24-MB01") {
    sku
    name
    custom_rating
  }
}
```

### Mutation Example
```graphql
mutation {
  addProductRating(input: {
    sku: "24-MB01"
    rating: 5
    comment: "Excellent product!"
  }) {
    success
    message
    rating {
      id
      rating
      comment
    }
  }
}
```

### With Variables
```graphql
query GetProducts($filter: ProductFilterInput, $pageSize: Int) {
  customProducts(filter: $filter, pageSize: $pageSize) {
    items {
      sku
      name
      custom_rating
    }
    total_count
  }
}
```

Variables:
```json
{
  "filter": {
    "min_rating": 4.0
  },
  "pageSize": 10
}
```

## Best Practices

1. **Use service contracts** in resolvers
2. **Follow GraphQL naming conventions** (camelCase fields)
3. **Document schema** with @doc annotations
4. **Handle errors properly** (use GraphQL exceptions)
5. **Implement pagination** for collections
6. **Check authorization** in mutations
7. **Use DataLoader pattern** to avoid N+1 queries
8. **Cache resolver results** when appropriate
9. **Version breaking changes** in schema
10. **Test with GraphQL IDE**

## Quality Bar

- Schema follows GraphQL best practices
- Resolvers use service contracts
- Proper error handling implemented
- Authorization checks in place
- Pagination implemented for collections
- Tests verify queries and mutations work
- Documentation in schema complete
