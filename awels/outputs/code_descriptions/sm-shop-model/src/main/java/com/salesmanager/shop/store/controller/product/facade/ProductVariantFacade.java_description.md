# ProductVariantFacade.java

## Review

## 1. Summary
- **Purpose**: Defines the contract for CRUD operations and listings on product variants in a shop application.  
- **Key components**:  
  - `ReadableProductVariant` – DTO for reading variant data.  
  - `PersistableProductVariant` – DTO for creating/updating variant data.  
  - `ReadableEntityList<ReadableProductVariant>` – paginated list wrapper.  
- **Frameworks/Libraries**: Relies on the Sales Manager core data models (`MerchantStore`, `Language`) but contains no framework‑specific annotations. It is a pure Java interface meant to be implemented by a service layer.

## 2. Detailed Description
The facade acts as an abstraction layer between the web/REST controllers and the underlying business logic (service, DAO, etc.).  
1. **Initialization** – Implementations typically inject a `ProductVariantService` or `ProductVariantDao` and map DTOs to entities.  
2. **Runtime behavior** –  
   - `get` retrieves a variant by its primary key, taking store and language into account for multi‑store/multi‑language contexts.  
   - `exists` checks SKU uniqueness within the store and product.  
   - `create` persists a new variant and returns its generated ID.  
   - `update` modifies an existing variant.  
   - `delete` removes a variant, usually cascading or updating references.  
   - `list` returns a paginated, language‑aware list of variants for a given product.  
3. **Cleanup** – None specified; implementations should manage transactions/close resources.

**Assumptions & Constraints**  
- `MerchantStore` and `Language` are non‑null and correctly authenticated.  
- The interface expects a one‑to‑one relationship between `productId` and variant operations.  
- Pagination parameters (`page`, `count`) are positive integers; negative or zero values should be handled by the implementation.

**Design Choices**  
- Using a facade decouples presentation from persistence.  
- DTOs (`PersistableProductVariant`/`ReadableProductVariant`) separate write‑from‑read concerns.  
- The method signatures reflect typical CRUD plus existence check and pagination.

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `ReadableProductVariant get(Long instanceId, Long productId, MerchantStore store, Language language)` | Retrieve a single variant by its ID. | `instanceId`: variant PK.<br>`productId`: parent product ID.<br>`store`: merchant context.<br>`language`: locale for localized fields. | Variant DTO or `null` if not found. | No state change. |
| `boolean exists(String sku, MerchantStore store, Long productId, Language language)` | Check if a SKU is already used within the given store and product. | `sku`: candidate SKU.<br>`store`: merchant context.<br>`productId`: parent product ID.<br>`language`: locale. | `true` if duplicate exists. | No state change. |
| `Long create(PersistableProductVariant productVariant, Long productId, MerchantStore store, Language language)` | Persist a new variant. | `productVariant`: data to store.<br>`productId`: parent product.<br>`store`, `language`. | Generated variant ID. | Writes to DB. |
| `void update(Long instanceId, PersistableProductVariant instance, Long productId, MerchantStore store, Language language)` | Update an existing variant. | `instanceId`: target variant.<br>`instance`: updated data.<br>`productId`, `store`, `language`. | None. | Modifies DB record. |
| `void delete(Long productVariant, Long productId, MerchantStore store)` | Remove a variant. | `productVariant`: target ID.<br>`productId`, `store`. | None. | Deletes DB record (may cascade). |
| `ReadableEntityList<ReadableProductVariant> list(Long productId, MerchantStore store, Language language, int page, int count)` | Retrieve paginated variants for a product. | `productId`, `store`, `language`, `page`, `count`. | Paginated list DTO. | No state change. |

*Reusable/Utility methods*: None within this interface; implementations may expose additional protected helpers.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.model.merchant.MerchantStore` | Core model | Represents the current merchant; no external libraries. |
| `com.salesmanager.core.model.reference.language.Language` | Core model | Locale information. |
| `com.salesmanager.shop.model.catalog.product.product.variant.PersistableProductVariant` | DTO | Write‑only representation. |
| `com.salesmanager.shop.model.catalog.product.product.variant.ReadableProductVariant` | DTO | Read‑only representation. |
| `com.salesmanager.shop.model.entity.ReadableEntityList` | DTO | Generic paginated list wrapper. |

All dependencies are internal to the Sales Manager platform and are part of the same project’s codebase. No third‑party libraries or framework annotations are referenced in this interface.

## 5. Additional Notes
- **Error handling**: The interface itself does not define exceptions. Implementations should document whether `null` or custom exceptions are used for missing records or validation failures.
- **Thread safety**: As an interface, no guarantees are made. Implementations should ensure that shared resources (e.g., caches, DAOs) are thread‑safe.
- **Security**: The methods rely on the caller providing a valid `MerchantStore`. Validation of store ownership and product ownership should be performed by the implementation to prevent cross‑store access.
- **Edge cases**:
  - Negative or zero `page`/`count` values.  
  - Very large `count` values could lead to memory issues.  
  - Deleting a variant that is referenced elsewhere (e.g., orders) should be handled carefully.
- **Future enhancements**:
  - Add bulk operations (`createBatch`, `deleteBatch`).  
  - Introduce filtering and sorting in `list` (e.g., by price, creation date).  
  - Support optimistic locking via version fields to prevent lost updates.  
  - Provide asynchronous methods or callbacks for long‑running operations.  

Overall, the interface is clean, concise, and follows good separation of concerns. Implementation details will determine the robustness of transaction management, caching, and error handling.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.product.facade;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.product.product.variant.PersistableProductVariant;
import com.salesmanager.shop.model.catalog.product.product.variant.ReadableProductVariant;
import com.salesmanager.shop.model.entity.ReadableEntityList;

public interface ProductVariantFacade {
	
	ReadableProductVariant get(Long instanceId, Long productId, MerchantStore store, Language language);
	boolean exists(String sku, MerchantStore store, Long productId, Language language);
	Long create(PersistableProductVariant productVariant, Long productId, MerchantStore store, Language language);
	void update(Long instanceId, PersistableProductVariant instance, Long productId, MerchantStore store, Language language);
	void delete(Long productVariant, Long productId, MerchantStore store);
	ReadableEntityList<ReadableProductVariant> list(Long productId, MerchantStore store, Language language, int page, int count);
	

}



```
