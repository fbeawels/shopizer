# ProductCommonFacade.java

## Review

## 1. Summary  
**Purpose** – The `ProductCommonFacade` interface defines a high‑level contract for managing products, product‑price, inventory, categories, and reviews in an e‑commerce shop.  It abstracts CRUD operations for both *persistable* (write‑side) and *readable* (view‑side) product representations and decouples the service layer from the persistence details.

**Key components**  
| Component | Role |
|-----------|------|
| `saveProduct` | Persist a new or updated product and return its identifier |
| `update` | Two overloads – update minimal product fields either by product id or SKU |
| `getProduct` / `getProductByCode` | Retrieve fully mapped `ReadableProduct` for a store‑specific language |
| `updateProductPrice` / `updateProductQuantity` | Adjust price or quantity for a product |
| `deleteProduct` | Remove a product by entity or by id+store |
| `addProductToCategory` / `removeProductFromCategory` | Manage category membership |
| `saveOrUpdateReview` / `deleteReview` / `getProductReviews` | CRUD and listing of product reviews |
| `exists` | Quick existence check by SKU and store |

The interface is intentionally free of implementation details, allowing multiple concrete facades (e.g., `ProductFacadeImpl`, `ProductFacadeRest`) to be swapped.  It relies on domain objects (`PersistableProduct`, `ReadableProduct`, etc.) rather than JPA entities to keep the service boundary clean.

## 2. Detailed Description  
1. **Execution Flow**  
   * The facade is typically injected into controller classes that handle HTTP requests.  
   * The controller passes a `MerchantStore` (representing the shop context) and a `Language` to all operations to enforce store‑ and locale‑specific behaviour.  
   * The facade translates DTOs (`PersistableProduct`, `LightPersistableProduct`) into internal `Product` entities, delegates to a service layer (e.g., `ProductService`), and then maps back to `ReadableProduct` DTOs for the view.

2. **Assumptions & Constraints**  
   * **Store isolation** – All methods require a `MerchantStore`; implementations must enforce that a product or review belongs to that store.  
   * **Locale awareness** – A `Language` is required for all read/write operations; implementations must translate between localized fields (e.g., names, descriptions).  
   * **Exception semantics** – The interface declares `throws Exception` for many methods; concrete implementations should either throw domain‑specific checked exceptions or convert to unchecked ones for clarity.  
   * **Concurrency** – Methods that modify data (e.g., `updateProductPrice`, `deleteProduct`) should consider optimistic locking or transaction boundaries.

3. **Architecture & Design Choices**  
   * **Facade pattern** – Encapsulates complex interactions between persistence, business rules, and DTO mapping.  
   * **DTO‑only interface** – No exposure of JPA entities; promotes API stability.  
   * **Overloading `update`** – Provides flexible updates by id or SKU, which can be useful for batch imports or SKU‑based APIs.  
   * **Language parameter** – A simple approach to localisation without needing a full i18n framework.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `Long saveProduct(MerchantStore store, PersistableProduct product, Language language)` | Persist or update a product; returns product ID. | Store, product DTO, language | Product ID | Inserts/updates DB, logs, triggers events |
| `void update(Long productId, LightPersistableProduct product, MerchantStore merchant, Language language)` | Update minimal product details by ID. | ID, light DTO, store, language | None | Update operation, audit trail |
| `void update(String sku, LightPersistableProduct product, MerchantStore merchant, Language language)` | Update minimal product details by SKU. | SKU, light DTO, store, language | None | Same as above |
| `ReadableProduct getProduct(MerchantStore store, Long id, Language language)` | Retrieve product by ID. | Store, ID, language | `ReadableProduct` | No DB mutation |
| `Product getProduct(Long id, MerchantStore store)` | Retrieve domain entity for internal use. | ID, store | `Product` | No DB mutation |
| `ReadableProduct getProductByCode(MerchantStore store, String uniqueCode, Language language)` | Retrieve product by unique code. | Store, code, language | `ReadableProduct` | No DB mutation |
| `ReadableProduct updateProductPrice(ReadableProduct product, ProductPriceEntity price, Language language)` | Set a new price for the given product. | Product DTO, price entity, language | Updated `ReadableProduct` | Update price in DB |
| `ReadableProduct updateProductQuantity(ReadableProduct product, int quantity, Language language)` | Set a new quantity for the given product. | Product DTO, quantity, language | Updated `ReadableProduct` | Update quantity in DB |
| `void deleteProduct(Product product)` | Delete product by entity. | `Product` | None | Remove from DB, cascade deletes |
| `void deleteProduct(Long id, MerchantStore store)` | Delete product by ID and store. | ID, store | None | Remove from DB |
| `ReadableProduct addProductToCategory(Category category, Product product, Language language)` | Link product to category. | Category entity, product entity, language | Updated `ReadableProduct` | Insert into join table |
| `ReadableProduct removeProductFromCategory(Category category, Product product, Language language)` | Unlink product from category. | Category, product, language | Updated `ReadableProduct` | Remove from join table |
| `void saveOrUpdateReview(PersistableProductReview review, MerchantStore store, Language language)` | Persist or update a review. | Review DTO, store, language | None | Insert/update review |
| `void deleteReview(ProductReview review, MerchantStore store, Language language)` | Delete a review. | Review entity, store, language | None | Remove from DB |
| `List<ReadableProductReview> getProductReviews(Product product, MerchantStore store, Language language)` | List all reviews for a product. | Product entity, store, language | List of `ReadableProductReview` | No DB mutation |
| `boolean exists(String sku, MerchantStore store)` | Quick existence check. | SKU, store | `true/false` | No DB mutation |

### Reusable / Utility Methods  
While this interface itself contains no static helpers, the `LightPersistableProduct` and `PersistableProductReview` DTOs are lightweight carriers intended for repeated use across multiple service layers.

## 4. Dependencies  

| Library / Framework | Purpose | Standard / Third‑party |
|---------------------|---------|------------------------|
| `com.salesmanager.core.model.*` | Domain entities for catalog, merchant, language, etc. | Third‑party (part of the SalesManager core) |
| `com.salesmanager.shop.model.*` | DTOs for shop layer | Third‑party |
| Java SE (java.util.List) | Standard collections | Standard |

The interface relies on the *SalesManager* domain model and DTOs; no direct Spring, JPA, or other dependency annotations are present, making the contract framework‑agnostic.

## 5. Additional Notes  

### Edge Cases & Missing Safety Nets  
1. **Null parameters** – The interface does not document or enforce non‑null contracts. Implementations must perform null‑checking or rely on framework validation (e.g., Spring’s `@Validated`).  
2. **Exception granularity** – Declaring `throws Exception` obscures specific error conditions (e.g., `ProductNotFoundException`, `DuplicateSkuException`). A richer exception hierarchy would aid callers.  
3. **Concurrency** – Methods that read and then modify (e.g., `updateProductPrice`) can suffer from lost updates if not wrapped in a transaction.  
4. **Bulk operations** – No batch update or delete methods; consider adding them for import/export scenarios.  
5. **Audit / event handling** – The interface does not expose hooks for auditing or domain events; implementations should be careful to propagate necessary metadata.

### Potential Enhancements  
- **Refactor `Language` usage** – Pass a `Locale` or `LocaleContext` to reduce coupling.  
- **Add default methods** – For common patterns such as “get by ID and throw if not found.”  
- **Introduce generic response wrapper** – Return a `Result<T>` that encapsulates success/failure, messages, and data.  
- **Versioning** – Annotate methods with API version tags if the interface will be exposed over HTTP.  
- **Security annotations** – Add `@PreAuthorize` or similar checks if the interface will be used by REST controllers.  
- **DTO validation** – Use Java Bean Validation annotations on DTO classes to enforce constraints before persistence.

Overall, the `ProductCommonFacade` is a clean, purpose‑driven contract that separates concerns between the shop API and the underlying business logic.  With modest tightening of exception handling, null safety, and some documentation of invariants, it would serve as a solid foundation for a robust product management module.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.product.facade;

import java.util.List;

import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.review.ProductReview;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.product.LightPersistableProduct;
import com.salesmanager.shop.model.catalog.product.PersistableProductReview;
import com.salesmanager.shop.model.catalog.product.ProductPriceEntity;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.catalog.product.ReadableProductReview;
import com.salesmanager.shop.model.catalog.product.product.PersistableProduct;

public interface ProductCommonFacade {
	

	  /**
	   * Create / Update product
	   * @param store
	   * @param product
	   * @param language
	   * @return
	   */
	  Long saveProduct(MerchantStore store, PersistableProduct product,
	      Language language);

	  /**
	   * Update minimal product details
	   * @param product
	   * @param merchant
	   * @param language
	   */
	  void update(Long productId, LightPersistableProduct product, MerchantStore merchant, Language language);
	  
	  /**
	   * Patch inventory by sku
	   * @param sku
	   * @param product
	   * @param merchant
	   * @param language
	   */
	  void update(String sku, LightPersistableProduct product, MerchantStore merchant, Language language);

	  /**
	   * Get a Product by id and store
	   *
	   * @param store
	   * @param id
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  ReadableProduct getProduct(MerchantStore store, Long id, Language language) throws Exception;
	  
	  /**
	   * 
	   * @param id
	   * @param store
	   * @return
	   */
	  Product getProduct(Long id, MerchantStore store);

	  /**
	   * Reads a product by code
	   *
	   * @param store
	   * @param uniqueCode
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  ReadableProduct getProductByCode(MerchantStore store, String uniqueCode, Language language)
	      throws Exception;


	  /**
	   * Sets a new price to an existing product
	   *
	   * @param product
	   * @param price
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  ReadableProduct updateProductPrice(ReadableProduct product, ProductPriceEntity price,
	      Language language) throws Exception;

	  /**
	   * Sets a new price to an existing product
	   *
	   * @param product
	   * @param quantity
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  ReadableProduct updateProductQuantity(ReadableProduct product, int quantity, Language language)
	      throws Exception;

	  /**
	   * Deletes a product for a given product id
	   *
	   * @param product
	   * @throws Exception
	   */
	  void deleteProduct(Product product) throws Exception;

	  /**
	   * Delete product
	   * @param id
	   * @param store
	   * @throws Exception
	   */
	  void deleteProduct(Long id, MerchantStore store);



	  /**
	   * Adds a product to a category
	   *
	   * @param category
	   * @param product
	   * @return
	   */
	  ReadableProduct addProductToCategory(Category category, Product product, Language language);

	  /**
	   * Removes item from a category
	   *
	   * @param category
	   * @param product
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  ReadableProduct removeProductFromCategory(Category category, Product product, Language language)
	      throws Exception;


	  /**
	   * Saves or updates a Product review
	   *
	   * @param review
	   * @param language
	   * @throws Exception
	   */
	  void saveOrUpdateReview(PersistableProductReview review, MerchantStore store, Language language)
	      throws Exception;

	  /**
	   * Deletes a product review
	   *
	   * @param review
	   * @param store
	   * @param language
	   * @throws Exception
	   */
	  void deleteReview(ProductReview review, MerchantStore store, Language language) throws Exception;

	  /**
	   * Get reviews for a given product
	   *
	   * @param product
	   * @param store
	   * @param language
	   * @return
	   * @throws Exception
	   */
	  List<ReadableProductReview> getProductReviews(Product product, MerchantStore store,
	      Language language) throws Exception;

	  /**
	   * validates if product exists
	   * @param sku
	   * @param store
	   * @return
	   */
	  public boolean exists(String sku, MerchantStore store);



}



```
