# ProductAvailabilityServiceImpl.java

## Review

## 1. Summary  

The `ProductAvailabilityServiceImpl` is a Spring Boot service that manages product availability (inventory) records.  
It extends a generic CRUD implementation (`SalesManagerEntityServiceImpl`) and adds several custom queries that are tailored to the business domain:

| Component | Role |
|-----------|------|
| `ProductAvailabilityRepository` | Basic JPA repository for CRUD operations. |
| `PageableProductAvailabilityRepository` | Custom repository that returns paged results for SKU‑ or product‑based queries. |
| `saveOrUpdate(ProductAvailability)` | Determines whether to create a new record or update an existing one. |
| `listByProduct(Long, MerchantStore, int, int)` | Paged list of availability records for a particular product. |
| `getById(Long, MerchantStore)` | Retrieve a single availability record by ID. |
| `getBySku(..)` overloads | Various paged or unpaged retrievals of availability records by SKU. |

The service relies heavily on Spring Data JPA (`Page`, `Pageable`, `PageRequest`) and uses Apache Commons `Validate` for argument checking.

## 2. Detailed Description  

### Architecture  
- **Base Service** – `SalesManagerEntityServiceImpl<Long, ProductAvailability>` supplies the standard CRUD operations (`create`, `update`, `delete`, etc.).  
- **Repository Layer** – Two JPA repositories are injected. The plain one offers default CRUD, while the pageable one exposes custom JPQL/HQL queries that return `Page<ProductAvailability>` results.  
- **Service Layer** – Implements business‑specific logic, primarily delegating to the repositories but also performing simple pre‑processing (validation, id checks).  

### Flow of Execution  

1. **Construction** – Spring injects the repositories through the constructor and field injection.  
2. **saveOrUpdate** –  
   - Checks if `availability.getId()` is a positive number.  
   - Calls `update()` (inherited from base) if the ID is present, otherwise `create()`.  
   - Returns the entity (now persisted).  
3. **Listing Methods** –  
   - Validate non‑null arguments.  
   - Build a `PageRequest`.  
   - Delegate to the pageable repository which executes a JPQL query and returns a `Page<ProductAvailability>`.  
4. **Retrieval Methods** –  
   - Validate arguments where necessary.  
   - Use the plain repository to fetch an entity or a list.  
   - Wrap single results in `Optional`.  

### Assumptions & Constraints  

- The underlying JPA provider (Hibernate, for example) is configured to auto‑handle transaction demarcation.  
- `ProductAvailabilityRepository.getById(Long)` is assumed to exist; most Spring Data JPA repos provide `findById` instead.  
- SKU and product codes are treated as case‑sensitive strings; no normalization is performed.  
- The service assumes that all IDs passed to `saveOrUpdate` are either `null` (create) or already persisted (update).  

### Design Choices  

- **Single Responsibility** – The service focuses on business rules; data access is delegated.  
- **Method Overloading** – The three `getBySku` methods allow callers to choose between paged and unpaged results.  
- **Optional Wrapper** – `getById` returns `Optional` to explicitly signal the possibility of a missing entity.  

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects | Notes |
|--------|---------|------------|---------|--------------|-------|
| `saveOrUpdate(ProductAvailability)` | Persist or update an availability record. | `availability` | `ProductAvailability` | Calls `create` or `update` which perform DB writes. | Checks `id` via `isPositive`. |
| `isPositive(Long)` | Helper to determine if an ID is a positive number. | `id` | `boolean` | None | Static‑like but instance method. |
| `listByProduct(Long, MerchantStore, int, int)` | Return paged availability for a specific product. | `productId`, `store`, `page`, `count` | `Page<ProductAvailability>` | None | Validates non‑null arguments. |
| `getById(Long, MerchantStore)` | Retrieve availability by ID. | `availabilityId`, `store` | `Optional<ProductAvailability>` | None | Uses repo `getById`. |
| `getBySku(String, MerchantStore, int, int)` | Paged retrieval by SKU for a store. | `sku`, `store`, `page`, `count` | `Page<ProductAvailability>` | None | Validates store. |
| `getBySku(String, int, int)` | Paged retrieval by SKU regardless of store. | `sku`, `page`, `count` | `Page<ProductAvailability>` | None | |
| `getBySku(String, MerchantStore)` | Retrieve all SKUs for a store. | `sku`, `store` | `List<ProductAvailability>` | None | Validates store. |

### Reusable / Utility Methods  

- `isPositive(Long)` could be made `static` and shared across services.  
- Validation logic (`Validate.notNull`) is repeated; a common argument‑validation helper could reduce duplication.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.stereotype.Service` | Spring Framework | Marks the class as a service component. |
| `org.springframework.data.domain.Page`, `Pageable`, `PageRequest` | Spring Data | For pagination support. |
| `javax.inject.Inject` | Java CDI / JSR‑330 | Alternative to Spring’s `@Autowired`. |
| `org.apache.commons.lang3.Validate` | Third‑party (Apache Commons) | Simple null / argument checks. |
| `com.salesmanager.core.business.repositories.*` | Project‑specific | JPA repositories for data access. |
| `com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl` | Project‑specific | Provides generic CRUD methods. |
| `com.salesmanager.core.model.*` | Project‑specific | Domain entities (`Product`, `ProductAvailability`, `MerchantStore`). |

All dependencies are either standard Spring Data JPA components or internal project artifacts; there are no external platform‑specific assumptions beyond a relational database.

## 5. Additional Notes  

### Edge Cases & Potential Issues  

1. **`getById` Method** – Uses `productAvailabilityRepository.getById(availabilityId)` which may throw an exception if the record is not found (depending on the repository implementation). It is wrapped in `Optional.ofNullable`, but if the underlying method throws, the exception propagates. Using `findById` would return an `Optional` safely.  

2. **Repository Method Names** – The custom repository interfaces (`PageableProductAvailabilityRepository`) are assumed to contain methods like `getByProductId` and `getBySku`. Their JPQL/SQL definitions are not shown; mis‑matching method signatures would cause runtime failures.  

3. **Validation Scope** – The service validates that `productId`, `store`, and `sku` are non‑null, but does not validate that `sku` is non‑empty or that `productId` > 0. A caller passing an invalid SKU string could result in an unexpected DB query.  

4. **Duplicate Handling** – `saveOrUpdate` simply decides between create or update based on ID presence. If a caller mistakenly provides an ID that does not exist in the database, `update` will attempt an update on a non‑existent record, potentially causing a `EntityNotFoundException` or silently failing.  

5. **Thread Safety** – The service is a singleton by default. All injected repositories are thread‑safe. No mutable shared state exists in the service itself, so concurrency is safe.  

### Potential Enhancements  

- **Centralize Validation** – Create a `ValidationUtils` class or use Spring’s `@Validated`/`@NotNull` annotations on method parameters.  
- **Use `findById`** – Replace `getById` with `findById` to avoid unchecked null dereferencing.  
- **DTO Layer** – Introduce Data Transfer Objects to decouple the API layer from the entity model, especially if SKU or product ID validation needs to be stricter.  
- **Logging** – Add SLF4J logging for CRUD operations and error cases to aid debugging.  
- **Exception Handling** – Wrap repository exceptions in a custom `ServiceException` to provide a unified error response for the service layer.  
- **Transaction Management** – Ensure that `saveOrUpdate` is annotated with `@Transactional` (if not handled by the base class) to guarantee atomicity.  

Overall, the implementation is straightforward and aligns with typical Spring Data JPA patterns. Minor adjustments around repository method usage, validation breadth, and error handling would increase robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.product.availability;

import java.util.List;
import java.util.Objects;
import java.util.Optional;

import javax.inject.Inject;

import org.apache.commons.lang3.Validate;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.catalog.product.availability.PageableProductAvailabilityRepository;
import com.salesmanager.core.business.repositories.catalog.product.availability.ProductAvailabilityRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.merchant.MerchantStore;

/**
 * Availability -> Inventory
 * 
 * @author carlsamson
 *
 */

@Service("productAvailabilityService")
public class ProductAvailabilityServiceImpl extends SalesManagerEntityServiceImpl<Long, ProductAvailability>
		implements ProductAvailabilityService {

	private ProductAvailabilityRepository productAvailabilityRepository;

	@Inject
	private PageableProductAvailabilityRepository pageableProductAvailabilityRepository;

	@Inject
	public ProductAvailabilityServiceImpl(ProductAvailabilityRepository productAvailabilityRepository) {
		super(productAvailabilityRepository);
		this.productAvailabilityRepository = productAvailabilityRepository;
	}

	@Override
	public ProductAvailability saveOrUpdate(ProductAvailability availability) throws ServiceException {
		if (isPositive(availability.getId())) {
			update(availability);
		} else {
			create(availability);
		}
		
		return availability;
	}

	private boolean isPositive(Long id) {
		return Objects.nonNull(id) && id > 0;
	}



	@Override
	public Page<ProductAvailability> listByProduct(Long productId, MerchantStore store, int page,
			int count) {
		Validate.notNull(productId, "Product cannot be null");
		Validate.notNull(store, "MercantStore cannot be null");
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableProductAvailabilityRepository.getByProductId(productId, store.getCode(), pageRequest);
	}



	@Override
	public Optional<ProductAvailability> getById(Long availabilityId, MerchantStore store) {
		Validate.notNull(store, "Merchant must not be null");
		return Optional.ofNullable(productAvailabilityRepository.getById(availabilityId));
	}

	@Override
	public Page<ProductAvailability> getBySku(String sku, MerchantStore store, int page, int count) {
		Validate.notNull(store, "MerchantStore cannot be null");
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableProductAvailabilityRepository.getBySku(sku, store.getCode(), pageRequest);
	}

	@Override
	public Page<ProductAvailability> getBySku(String sku, int page, int count) {
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableProductAvailabilityRepository.getBySku(sku, pageRequest);
	}

	@Override
	public List<ProductAvailability> getBySku(String sku, MerchantStore store) {
		Validate.notNull(store, "MerchantStore cannot be null");
		return productAvailabilityRepository.getBySku(sku, store.getCode());
	}

}



```
