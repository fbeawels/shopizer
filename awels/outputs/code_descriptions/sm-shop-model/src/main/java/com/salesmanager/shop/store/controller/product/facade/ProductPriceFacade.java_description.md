# ProductPriceFacade.java

## Review

## 1. Summary
The `ProductPriceFacade` interface defines the contract for managing product pricing within the SalesManager e‑commerce platform. It exposes CRUD‑style operations for creating, retrieving, listing, and deleting product prices, while abstracting away the underlying persistence and business logic. The interface is intended to be implemented by a concrete service that interacts with the core data layer (`PersistableProductPrice`, `ReadableProductPrice`, `MerchantStore`, `Language`, etc.).  

**Key components**  
- **PersistableProductPrice** – DTO used to create or update a product price.  
- **ReadableProductPrice** – DTO used to expose price data to callers (e.g., REST endpoints).  
- **MerchantStore** – Contextual information about the store in which the operation is performed.  
- **Language** – Locale used for localised fields such as descriptions or currency symbols.  

The interface follows a typical **Facade** design pattern: it hides the complexity of the underlying services and data access objects, presenting a simplified API to callers (e.g., controllers, other facades).

## 2. Detailed Description
### Core Flow
1. **Create (`save`)** – Accepts a `PersistableProductPrice` and a `MerchantStore`, persists the entity, and returns the generated primary key.  
2. **Delete (`delete`)** – Removes a price record identified by `priceId` and associated with a specific product SKU and store.  
3. **List (`list`)** – Two overloads allow listing by SKU only, or by SKU and inventory ID (for variants). A `Language` parameter enables localisation of returned price data.  
4. **Get (`get`)** – Retrieves a single `ReadableProductPrice` for a given SKU, price ID, store, and language.

The interface is deliberately thin; implementation responsibilities include:
- Validating business rules (e.g., price > 0, inventory exists).  
- Translating between entity objects and DTOs.  
- Handling localisation (currency formatting, language-specific fields).  
- Managing transactional boundaries.

### Assumptions & Constraints
- **SKU uniqueness** – All methods assume that `sku` uniquely identifies a product within the given store context.  
- **Inventory optional** – For listings, `inventoryId` may be null, implying a request for all variant prices.  
- **Transactional integrity** – The interface itself does not enforce transactions; implementation must manage them.  
- **Localization** – The `Language` object is required to fetch or format data in the correct locale.  

### Architecture
The facade sits between the web layer (controllers, services) and the persistence layer (DAOs, repositories). It promotes separation of concerns and facilitates unit testing by allowing mocking of the facade in higher‑level components.

## 3. Functions/Methods
| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `Long save(PersistableProductPrice price, MerchantStore store)` | Persist a new product price and return its ID. | Create operation. | `price`: data to persist.<br>`store`: context of the store. | Generated `Long` primary key. | Persists a record; may trigger cascades. |
| `void delete(Long priceId, String sku, MerchantStore store)` | Remove a price record. | Delete operation. | `priceId`: primary key of the price.<br>`sku`: product identifier.<br>`store`: context. | None. | Deletes record; may cascade deletions. |
| `List<ReadableProductPrice> list(String sku, Long inventoryId, MerchantStore store, Language language)` | Retrieve all prices for a specific SKU & inventory. | Read‑many. | `sku`, `inventoryId`, `store`, `language`. | List of `ReadableProductPrice`. | None. |
| `List<ReadableProductPrice> list(String sku, MerchantStore store, Language language)` | Retrieve all prices for a SKU across all inventories. | Read‑many. | `sku`, `store`, `language`. | List of `ReadableProductPrice`. | None. |
| `ReadableProductPrice get(String sku, Long productPriceId, MerchantStore store, Language language)` | Retrieve a single price by ID. | Read‑one. | `sku`, `productPriceId`, `store`, `language`. | `ReadableProductPrice` or `null`. | None. |

### Reusable/Utility Methods
While the interface itself does not provide utilities, implementations often expose helper methods for:
- **Validation** (e.g., `validatePrice(PersistableProductPrice)`).
- **DTO mapping** (`toReadable(PersistableProductPrice)`).
- **Locale formatting**.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.model.merchant.MerchantStore` | Core model | Represents store context; part of SalesManager core library. |
| `com.salesmanager.core.model.reference.language.Language` | Core model | Holds locale information; part of core reference data. |
| `com.salesmanager.shop.model.catalog.product.PersistableProductPrice` | Shop DTO | Holds input data for price creation. |
| `com.salesmanager.shop.model.catalog.product.ReadableProductPrice` | Shop DTO | Holds output data for price queries. |
| Java Collections (`java.util.List`) | Standard | Basic data structure. |

All dependencies are either **core SalesManager modules** (not third‑party) or standard Java libraries. There are no platform‑specific or optional dependencies visible in this interface.

## 5. Additional Notes
### Strengths
- **Clean separation**: The interface is concise and focuses purely on price management, making it easy to mock and test.  
- **Localization support**: Explicit `Language` parameter allows for future expansions (e.g., multi‑currency).  
- **Extensibility**: New methods (e.g., bulk update) can be added without affecting existing clients.

### Potential Issues & Edge Cases
- **SKU validation** – The interface assumes `sku` is always valid and present; a null or empty SKU could lead to NullPointerExceptions in implementations.  
- **Null handling** – Methods return raw `List` or `ReadableProductPrice`. Implementations should decide whether to return empty lists or throw exceptions for missing data.  
- **Concurrency** – `save` and `delete` may conflict if concurrent updates occur; transactional isolation must be enforced.  
- **Inventory ID nullability** – In `list(sku, inventoryId, ...)`, passing `null` could cause ambiguous queries; implementation must clearly document the behavior.  

### Future Enhancements
1. **Pagination & Sorting** – Add parameters for page number, page size, and sort order to `list` methods.  
2. **Bulk Operations** – `saveAll(List<PersistableProductPrice>)` and `deleteAll(List<Long>)` could improve performance.  
3. **Price Versioning** – Track historical price changes with effective dates.  
4. **Exception Hierarchy** – Define domain‑specific exceptions (e.g., `PriceNotFoundException`, `InvalidPriceException`) for clearer error handling.  
5. **Asynchronous Support** – Return `CompletableFuture` or similar for non‑blocking operations in reactive architectures.  

Overall, the `ProductPriceFacade` interface provides a solid foundation for product price management. The real value will come from a robust implementation that enforces business rules, handles localization, and integrates cleanly with the rest of the SalesManager ecosystem.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.product.facade;

import java.util.List;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.product.PersistableProductPrice;
import com.salesmanager.shop.model.catalog.product.ReadableProductPrice;


/**
 * Product price management api
 * @author carlsamson
 *
 */
public interface ProductPriceFacade {

	/**
	 * Calculate product price based on specific product options
	 * 
	 * @param id
	 * @param priceRequest
	 * @param store
	 * @param language
	 * @return
	 */
	/**
	 * ReadableProductPrice getProductPrice(Long id, ProductPriceRequest
	 * priceRequest, MerchantStore store, Language language);
	 **/

	/**
	 * Creates a product price
	 * @param price
	 * @param productId
	 * @param inventoryId
	 * @param store
	 * @return
	 */
	Long save(PersistableProductPrice price, MerchantStore store);

	/**
	 * Product price deletion
	 * @param priceId
	 * @param productId
	 * @param inventoryId
	 * @param store
	 */
	void delete(Long priceId, String sku, MerchantStore store);
	
	/**
	 * List product prices by product and inventory (product and variants)
	 * @param productId
	 * @param inventoryId
	 * @param store
	 * @return
	 */
	List<ReadableProductPrice> list(String sku, Long inventoryId, MerchantStore store, Language language);
	
	/**
	 * List product prices by product
	 * @param poductId
	 * @param store
	 * @return
	 */
	List<ReadableProductPrice> list(String sku, MerchantStore store, Language language);
	
	/**
	 * Get ProductPrice
	 * @param sku
	 * @param productPriceId
	 * @param store
	 * @param language
	 * @return
	 */
	ReadableProductPrice get(String sku, Long productPriceId, MerchantStore store, Language language);
}



```
