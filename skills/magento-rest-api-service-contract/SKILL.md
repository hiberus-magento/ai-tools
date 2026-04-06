---
name: magento-rest-api-service-contract
description: Designs rigorous PHP interfaces and exposes them as REST API endpoints through webapi.xml for secure ERP/CRM integrations. Use when building API integrations, exposing custom functionality, or creating machine-to-machine communication. Requires Magento 2 API knowledge.
allowed-tools: Read Grep Glob Bash Edit Write
model: opus
---

# Magento REST API Service Contract

Creates service contract interfaces and exposes them as REST API endpoints via `webapi.xml`, enabling secure integrations with ERP, CRM, PIM, and other external systems following Magento 2 best practices.

## When to Use

- Building custom API endpoints
- ERP/PIM/CRM integrations
- Third-party system communication
- Mobile app backends
- B2B partner integrations
- Webhook implementations
- Microservices architecture

## Service Contract Architecture

```
Interface (API contract)
    ↓
Implementation (Business logic)
    ↓
webapi.xml (REST/SOAP exposure)
    ↓
ACL (Authorization)
```

## Execution Workflow

1. **Define API requirements**
   - Identify operations needed
   - Define input/output data structures
   - Plan authentication/authorization
   - Consider versioning strategy

2. **Create data interfaces**
   - Define DTOs (Data Transfer Objects)
   - Use getters/setters
   - Follow SOLID principles

3. **Create service interface**
   - Define method signatures
   - Use type hints
   - Document with PHPDoc

4. **Implement service**
   - Business logic in implementation class
   - Use dependency injection
   - Handle exceptions properly

5. **Expose via webapi.xml**
   - Map interface to REST route
   - Define HTTP method
   - Set ACL resources

6. **Configure ACL**
   - Define permissions in acl.xml
   - Set appropriate resource hierarchy

7. **Test API endpoints**
   - Use Postman/cURL
   - Test authentication
   - Verify responses

## Implementation Pattern

### Step 1: Data Interface (DTO)

**File:** `app/code/Vendor/Module/Api/Data/RatingInterface.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api\Data;

interface RatingInterface
{
    const RATING_ID = 'rating_id';
    const CUSTOMER_ID = 'customer_id';
    const PRODUCT_SKU = 'product_sku';
    const RATING = 'rating';
    const COMMENT = 'comment';
    const CREATED_AT = 'created_at';
    
    /**
     * Get rating ID
     *
     * @return int|null
     */
    public function getRatingId(): ?int;
    
    /**
     * Set rating ID
     *
     * @param int $ratingId
     * @return $this
     */
    public function setRatingId(int $ratingId): self;
    
    /**
     * Get customer ID
     *
     * @return int
     */
    public function getCustomerId(): int;
    
    /**
     * Set customer ID
     *
     * @param int $customerId
     * @return $this
     */
    public function setCustomerId(int $customerId): self;
    
    /**
     * Get product SKU
     *
     * @return string
     */
    public function getProductSku(): string;
    
    /**
     * Set product SKU
     *
     * @param string $sku
     * @return $this
     */
    public function setProductSku(string $sku): self;
    
    /**
     * Get rating value
     *
     * @return int
     */
    public function getRating(): int;
    
    /**
     * Set rating value
     *
     * @param int $rating
     * @return $this
     */
    public function setRating(int $rating): self;
    
    /**
     * Get comment
     *
     * @return string|null
     */
    public function getComment(): ?string;
    
    /**
     * Set comment
     *
     * @param string|null $comment
     * @return $this
     */
    public function setComment(?string $comment): self;
    
    /**
     * Get created at timestamp
     *
     * @return string|null
     */
    public function getCreatedAt(): ?string;
    
    /**
     * Set created at timestamp
     *
     * @param string $createdAt
     * @return $this
     */
    public function setCreatedAt(string $createdAt): self;
}
```

### Step 2: Data Model Implementation

**File:** `app/code/Vendor/Module/Model/Rating.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;
use Vendor\Module\Api\Data\RatingInterface;

class Rating extends AbstractModel implements RatingInterface
{
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Rating::class);
    }
    
    public function getRatingId(): ?int
    {
        return $this->getData(self::RATING_ID) 
            ? (int) $this->getData(self::RATING_ID) 
            : null;
    }
    
    public function setRatingId(int $ratingId): RatingInterface
    {
        return $this->setData(self::RATING_ID, $ratingId);
    }
    
    public function getCustomerId(): int
    {
        return (int) $this->getData(self::CUSTOMER_ID);
    }
    
    public function setCustomerId(int $customerId): RatingInterface
    {
        return $this->setData(self::CUSTOMER_ID, $customerId);
    }
    
    public function getProductSku(): string
    {
        return (string) $this->getData(self::PRODUCT_SKU);
    }
    
    public function setProductSku(string $sku): RatingInterface
    {
        return $this->setData(self::PRODUCT_SKU, $sku);
    }
    
    public function getRating(): int
    {
        return (int) $this->getData(self::RATING);
    }
    
    public function setRating(int $rating): RatingInterface
    {
        return $this->setData(self::RATING, $rating);
    }
    
    public function getComment(): ?string
    {
        return $this->getData(self::COMMENT);
    }
    
    public function setComment(?string $comment): RatingInterface
    {
        return $this->setData(self::COMMENT, $comment);
    }
    
    public function getCreatedAt(): ?string
    {
        return $this->getData(self::CREATED_AT);
    }
    
    public function setCreatedAt(string $createdAt): RatingInterface
    {
        return $this->setData(self::CREATED_AT, $createdAt);
    }
}
```

### Step 3: Service Interface

**File:** `app/code/Vendor/Module/Api/RatingRepositoryInterface.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api;

use Vendor\Module\Api\Data\RatingInterface;
use Magento\Framework\Api\SearchCriteriaInterface;
use Magento\Framework\Api\SearchResultsInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;

interface RatingRepositoryInterface
{
    /**
     * Save rating
     *
     * @param RatingInterface $rating
     * @return RatingInterface
     * @throws LocalizedException
     */
    public function save(RatingInterface $rating): RatingInterface;
    
    /**
     * Get rating by ID
     *
     * @param int $ratingId
     * @return RatingInterface
     * @throws NoSuchEntityException
     */
    public function getById(int $ratingId): RatingInterface;
    
    /**
     * Get rating by product SKU and customer ID
     *
     * @param string $sku
     * @param int $customerId
     * @return RatingInterface
     * @throws NoSuchEntityException
     */
    public function getBySkuAndCustomer(string $sku, int $customerId): RatingInterface;
    
    /**
     * Get list of ratings
     *
     * @param SearchCriteriaInterface $searchCriteria
     * @return SearchResultsInterface
     */
    public function getList(SearchCriteriaInterface $searchCriteria): SearchResultsInterface;
    
    /**
     * Delete rating
     *
     * @param RatingInterface $rating
     * @return bool
     * @throws LocalizedException
     */
    public function delete(RatingInterface $rating): bool;
    
    /**
     * Delete rating by ID
     *
     * @param int $ratingId
     * @return bool
     * @throws NoSuchEntityException
     * @throws LocalizedException
     */
    public function deleteById(int $ratingId): bool;
}
```

### Step 4: Service Implementation

**File:** `app/code/Vendor/Module/Model/RatingRepository.php`
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Vendor\Module\Api\RatingRepositoryInterface;
use Vendor\Module\Api\Data\RatingInterface;
use Vendor\Module\Api\Data\RatingInterfaceFactory;
use Vendor\Module\Model\ResourceModel\Rating as RatingResource;
use Vendor\Module\Model\ResourceModel\Rating\CollectionFactory;
use Magento\Framework\Api\SearchCriteriaInterface;
use Magento\Framework\Api\SearchResultsInterfaceFactory;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Framework\Exception\CouldNotSaveException;
use Magento\Framework\Exception\CouldNotDeleteException;

class RatingRepository implements RatingRepositoryInterface
{
    private RatingResource $resource;
    private RatingInterfaceFactory $ratingFactory;
    private CollectionFactory $collectionFactory;
    private SearchResultsInterfaceFactory $searchResultsFactory;
    
    public function __construct(
        RatingResource $resource,
        RatingInterfaceFactory $ratingFactory,
        CollectionFactory $collectionFactory,
        SearchResultsInterfaceFactory $searchResultsFactory
    ) {
        $this->resource = $resource;
        $this->ratingFactory = $ratingFactory;
        $this->collectionFactory = $collectionFactory;
        $this->searchResultsFactory = $searchResultsFactory;
    }
    
    public function save(RatingInterface $rating): RatingInterface
    {
        try {
            $this->resource->save($rating);
        } catch (\Exception $e) {
            throw new CouldNotSaveException(__('Could not save rating: %1', $e->getMessage()));
        }
        return $rating;
    }
    
    public function getById(int $ratingId): RatingInterface
    {
        $rating = $this->ratingFactory->create();
        $this->resource->load($rating, $ratingId);
        
        if (!$rating->getRatingId()) {
            throw new NoSuchEntityException(__('Rating with ID "%1" does not exist', $ratingId));
        }
        
        return $rating;
    }
    
    public function getBySkuAndCustomer(string $sku, int $customerId): RatingInterface
    {
        $collection = $this->collectionFactory->create();
        $collection->addFieldToFilter('product_sku', $sku)
                   ->addFieldToFilter('customer_id', $customerId);
        
        $rating = $collection->getFirstItem();
        
        if (!$rating->getRatingId()) {
            throw new NoSuchEntityException(
                __('Rating for SKU "%1" and customer "%2" does not exist', $sku, $customerId)
            );
        }
        
        return $rating;
    }
    
    public function getList(SearchCriteriaInterface $searchCriteria): \Magento\Framework\Api\SearchResultsInterface
    {
        $collection = $this->collectionFactory->create();
        
        foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
            foreach ($filterGroup->getFilters() as $filter) {
                $collection->addFieldToFilter(
                    $filter->getField(),
                    [$filter->getConditionType() => $filter->getValue()]
                );
            }
        }
        
        $searchResults = $this->searchResultsFactory->create();
        $searchResults->setSearchCriteria($searchCriteria);
        $searchResults->setItems($collection->getItems());
        $searchResults->setTotalCount($collection->getSize());
        
        return $searchResults;
    }
    
    public function delete(RatingInterface $rating): bool
    {
        try {
            $this->resource->delete($rating);
        } catch (\Exception $e) {
            throw new CouldNotDeleteException(__('Could not delete rating: %1', $e->getMessage()));
        }
        return true;
    }
    
    public function deleteById(int $ratingId): bool
    {
        return $this->delete($this->getById($ratingId));
    }
}
```

### Step 5: Configure DI (Preference)

**File:** `app/code/Vendor/Module/etc/di.xml`
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    
    <!-- Map interface to implementation -->
    <preference for="Vendor\Module\Api\RatingRepositoryInterface" 
                type="Vendor\Module\Model\RatingRepository"/>
    
    <preference for="Vendor\Module\Api\Data\RatingInterface" 
                type="Vendor\Module\Model\Rating"/>
</config>
```

### Step 6: Expose REST API

**File:** `app/code/Vendor/Module/etc/webapi.xml`
```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">
    
    <!-- POST /V1/ratings -->
    <route url="/V1/ratings" method="POST">
        <service class="Vendor\Module\Api\RatingRepositoryInterface" method="save"/>
        <resources>
            <resource ref="Vendor_Module::rating_save"/>
        </resources>
    </route>
    
    <!-- GET /V1/ratings/:id -->
    <route url="/V1/ratings/:ratingId" method="GET">
        <service class="Vendor\Module\Api\RatingRepositoryInterface" method="getById"/>
        <resources>
            <resource ref="Vendor_Module::rating_view"/>
        </resources>
    </route>
    
    <!-- GET /V1/ratings/search -->
    <route url="/V1/ratings" method="GET">
        <service class="Vendor\Module\Api\RatingRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Vendor_Module::rating_view"/>
        </resources>
    </route>
    
    <!-- GET /V1/ratings/sku/:sku/customer/:customerId -->
    <route url="/V1/ratings/sku/:sku/customer/:customerId" method="GET">
        <service class="Vendor\Module\Api\RatingRepositoryInterface" method="getBySkuAndCustomer"/>
        <resources>
            <resource ref="Vendor_Module::rating_view"/>
        </resources>
    </route>
    
    <!-- DELETE /V1/ratings/:id -->
    <route url="/V1/ratings/:ratingId" method="DELETE">
        <service class="Vendor\Module\Api\RatingRepositoryInterface" method="deleteById"/>
        <resources>
            <resource ref="Vendor_Module::rating_delete"/>
        </resources>
    </route>
    
</routes>
```

### Step 7: Define ACL

**File:** `app/code/Vendor/Module/etc/acl.xml`
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="Vendor_Module::rating" title="Ratings" sortOrder="100">
                    <resource id="Vendor_Module::rating_view" title="View Ratings" sortOrder="10"/>
                    <resource id="Vendor_Module::rating_save" title="Create/Update Ratings" sortOrder="20"/>
                    <resource id="Vendor_Module::rating_delete" title="Delete Ratings" sortOrder="30"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

## API Authentication

### Admin Token
```bash
curl -X POST "https://magento.local/rest/V1/integration/admin/token" \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"admin123"}'
```

### Customer Token
```bash
curl -X POST "https://magento.local/rest/V1/integration/customer/token" \
     -H "Content-Type: application/json" \
     -d '{"username":"customer@example.com","password":"password"}'
```

## API Usage Examples

### Create Rating (POST)
```bash
curl -X POST "https://magento.local/rest/V1/ratings" \
     -H "Authorization: Bearer {token}" \
     -H "Content-Type: application/json" \
     -d '{
  "rating": {
    "customer_id": 1,
    "product_sku": "24-MB01",
    "rating": 5,
    "comment": "Excellent product!"
  }
}'
```

### Get Rating by ID (GET)
```bash
curl -X GET "https://magento.local/rest/V1/ratings/1" \
     -H "Authorization: Bearer {token}"
```

### Search Ratings (GET with SearchCriteria)
```bash
curl -X GET "https://magento.local/rest/V1/ratings?searchCriteria[filterGroups][0][filters][0][field]=product_sku&searchCriteria[filterGroups][0][filters][0][value]=24-MB01&searchCriteria[filterGroups][0][filters][0][conditionType]=eq" \
     -H "Authorization: Bearer {token}"
```

### Delete Rating (DELETE)
```bash
curl -X DELETE "https://magento.local/rest/V1/ratings/1" \
     -H "Authorization: Bearer {token}"
```

## Best Practices

1. **Use service contracts** (interfaces) not concrete classes
2. **Follow SOLID principles**
3. **Validate input** in implementation
4. **Use proper HTTP status codes**
5. **Document API** with PHPDoc
6. **Version your APIs** (/V1/, /V2/)
7. **Implement SearchCriteria** for list operations
8. **Use ACL** for fine-grained permissions
9. **Test with Postman/Swagger**
10. **Log API errors** appropriately

## Quality Bar

- Service contracts well-defined
- Implementation follows Magento standards
- webapi.xml correctly maps routes
- ACL resources properly defined
- Authentication working
- API responses follow conventions
- Error handling implemented
- Tests verify functionality
