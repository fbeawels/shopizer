# CategoryFacade.java

## Review

## 1. Summary

The `CategoryFacade` interface defines a contract for managing and retrieving category data in the Sales Manager e‑commerce system. It provides methods for CRUD operations, hierarchy navigation, product variant lookup, visibility toggling, and other domain‑specific actions. The facade hides the underlying persistence and business logic layers from the controller or API layer, enabling a clean separation of concerns.

Key components:
- **CRUD & Retrieval** – `saveCategory`, `getById`, `getByCode`, `getCategoryByFriendlyUrl`, `deleteCategory`, `existByCode`.
- **Hierarchy Operations** – `getCategoryHierarchy`, `move`.
- **Visibility Control** – `setVisible`.
- **Product Variant & Association** – `categoryProductVariants`, `listByProduct`.

The code uses standard Java collections and custom domain models (`PersistableCategory`, `ReadableCategory`, etc.), and relies on `MerchantStore` and `Language` objects to provide store‑specific and localized context.

## 2. Detailed Description

### Core Flow

1. **Data Access Layer**  
   The facade sits on top of one or more service or repository layers. Concrete implementations will inject the necessary services (e.g., `CategoryService`, `ProductService`) to perform persistence and business logic.

2. **Hierarchical Retrieval**  
   `getCategoryHierarchy` is intended to return a nested structure of categories up to a specified depth. It takes a `ListCriteria` (pagination, sorting, etc.) and optional `filter` list for additional constraints.

3. **Persistence**  
   `saveCategory` receives a `PersistableCategory` (likely a DTO containing new or updated data) and returns the persisted entity. `deleteCategory` overloads allow deletion by id or entity.

4. **Lookup**  
   `getById`, `getByCode`, and `getCategoryByFriendlyUrl` provide read‑only views (`ReadableCategory`) for various key attributes.

5. **Business Rules**  
   - `setVisible` toggles the visibility flag on a category.  
   - `move` changes a category’s parent node, which may trigger re‑ordering or tree re‑construction.  
   - `existByCode` validates uniqueness of a category code.

6. **Product Associations**  
   `categoryProductVariants` retrieves all variants of products belonging to a category, while `listByProduct` lists categories associated with a particular product.

### Assumptions & Constraints

- **Thread Safety** – Implementations must be thread‑safe as facades can be used in a multi‑threaded web context.  
- **Null Handling** – No explicit null checks in the interface; implementations should decide on null/empty semantics.  
- **Exceptions** – Only `getByCode` and `getCategoryByFriendlyUrl` declare `throws Exception`; the rest rely on runtime exceptions.  
- **Language & Store Context** – All methods require both a `MerchantStore` and a `Language`, implying that categories are store‑specific and multi‑lingual.

### Architecture

The interface follows the **Facade** pattern, providing a simplified API over potentially complex service layers. It also aligns with **Domain‑Driven Design (DDD)** principles: domain objects (`Category`, `PersistableCategory`) and context objects (`MerchantStore`, `Language`) drive the interface contract.

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects | Remarks |
|--------|---------|--------|---------|--------------|---------|
| `ReadableCategoryList getCategoryHierarchy(MerchantStore, ListCriteria, int, Language, List<String>, int, int)` | Retrieve a hierarchical list of categories, depth‑controlled, paginated. | Store, pagination/sort (`ListCriteria`), depth, language, optional filters, page, count | `ReadableCategoryList` (nested categories) | None (read‑only) | Requires efficient tree traversal; likely uses `LEFT/RIGHT` bounds or adjacency list. |
| `PersistableCategory saveCategory(MerchantStore, PersistableCategory)` | Persist or update a category. | Store, category DTO | Persisted `PersistableCategory` (with ID, timestamps) | Database write | Validation of uniqueness, code format expected. |
| `ReadableCategory getById(MerchantStore, Long, Language)` | Fetch category by primary key. | Store, id, language | `ReadableCategory` | None | Must handle missing id → throw or return null. |
| `ReadableCategory getByCode(MerchantStore, String, Language) throws Exception` | Fetch category by code. | Store, code, language | `ReadableCategory` | None | Throws checked exception on failure. |
| `ReadableCategory getCategoryByFriendlyUrl(MerchantStore, String, Language) throws Exception` | Fetch category by SEO friendly URL. | Store, friendlyUrl, language | `ReadableCategory` | None | Used by routing to resolve URLs. |
| `Category getByCode(String, MerchantStore)` | Variant that returns domain `Category` instead of readable DTO. | Code, store | `Category` | None | Might be used internally by services. |
| `void deleteCategory(Long, MerchantStore)` | Delete by ID. | ID, store | N/A | Remove record from DB | Cascade delete for child categories may be needed. |
| `void deleteCategory(Category)` | Delete by entity. | Category | N/A | Same as above | Convenience overload. |
| `List<ReadableProductVariant> categoryProductVariants(Long, MerchantStore, Language)` | List product variants belonging to a category. | Category ID, store, language | List of variants | None | May perform joins between product, variant, and category tables. |
| `boolean existByCode(MerchantStore, String)` | Check code uniqueness. | Store, code | true/false | None | Should be fast (index on code). |
| `void move(Long, Long, MerchantStore)` | Re‑parent a category. | Child ID, new parent ID, store | N/A | Adjust tree structure | Must maintain integrity, update `LEFT/RIGHT` bounds. |
| `void setVisible(PersistableCategory, MerchantStore)` | Toggle visibility flag. | Category DTO, store | N/A | Update visibility flag in DB | May trigger cache invalidation. |
| `ReadableCategoryList listByProduct(MerchantStore, Long, Language)` | Retrieve categories associated with a product. | Store, product ID, language | `ReadableCategoryList` | None | Useful for breadcrumbs or navigation. |

**Reusable/Utility Methods**  
None are defined in the interface, but the methods above can be combined by concrete implementations to expose higher‑level operations.

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.model.catalog.category.Category` | Domain model | Entity representing a category. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Domain model | Stores the context (store ID, locale). |
| `com.salesmanager.core.model.reference.language.Language` | Domain model | Language code. |
| `com.salesmanager.shop.model.catalog.category.PersistableCategory` | DTO | Input for create/update operations. |
| `com.salesmanager.shop.model.catalog.category.ReadableCategory` | DTO | Read‑only representation for API consumers. |
| `com.salesmanager.shop.model.catalog.category.ReadableCategoryList` | Collection DTO | Represents hierarchical or paginated lists. |
| `com.salesmanager.shop.model.catalog.product.attribute.ReadableProductVariant` | DTO | Represents product variants. |
| `com.salesmanager.shop.model.entity.ListCriteria` | DTO | Encapsulates pagination, sorting, filtering. |

All are **internal (Sales Manager) libraries**, not external third‑party dependencies. The interface assumes the presence of a persistence framework (likely JPA/Hibernate) and a transactional environment (e.g., Spring).

## 5. Additional Notes

### Edge Cases & Error Handling
- **Missing or Null Parameters** – Implementations should validate inputs; the interface does not enforce contracts.
- **Concurrency** – Simultaneous `move` or `delete` operations could corrupt the category tree; proper locking or optimistic concurrency control is essential.
- **Cycle Prevention** – The `move` operation must guard against creating cycles (a node becoming its own ancestor).
- **Language Fallback** – If a category has no translation for the requested language, should fallback to default or throw an exception.
- **Pagination Bounds** – `getCategoryHierarchy` accepts `page` and `count`; boundary checks (negative values) are required.
- **Exception Propagation** – Only two methods declare checked exceptions; the rest rely on runtime. Consistency would improve API clarity.

### Potential Enhancements
- **Return Optional** – Use `Optional<ReadableCategory>` for `getById`/`getByCode` to explicitly handle missing results.
- **Batch Operations** – Add methods for bulk delete or bulk update visibility.
- **Search & Filtering** – Extend `ListCriteria` to include category name or slug filters.
- **Cache Integration** – Expose cache invalidation hooks or integrate with existing cache layers.
- **Audit Fields** – Include `createdBy`, `updatedBy` metadata in DTOs.
- **DTO Versioning** – Separate read/write DTOs more clearly; consider using projections for read‑only views.

Overall, the `CategoryFacade` interface is cleanly designed and aligns with typical e‑commerce domain requirements. Its clarity facilitates unit testing and makes the service layer’s responsibilities explicit. The main improvements revolve around defensive programming, consistency in error handling, and richer API expressiveness for edge conditions.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.category.facade;

import java.util.List;
import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.category.PersistableCategory;
import com.salesmanager.shop.model.catalog.category.ReadableCategory;
import com.salesmanager.shop.model.catalog.category.ReadableCategoryList;
import com.salesmanager.shop.model.catalog.product.attribute.ReadableProductVariant;
import com.salesmanager.shop.model.entity.ListCriteria;

public interface CategoryFacade {


    /**
     * Returns a list of ReadableCategory ordered and built according to a given depth
     * @param store
     * @param depth
     * @param language
     * @param filter
     * @param page
     * @param count
     * @return ReadableCategoryList
     */
	ReadableCategoryList getCategoryHierarchy(MerchantStore store, ListCriteria criteria, int depth, Language language, List<String> filter, int page, int count);

	/**
	 *
	 * @param store
	 * @param category
	 * @return PersistableCategory
	 */
	PersistableCategory saveCategory(MerchantStore store, PersistableCategory category);

	/**
	 *
	 * @param store
	 * @param id
	 * @param language
	 * @return ReadableCategory
	 */
	ReadableCategory getById(MerchantStore store, Long id, Language language);

	/**
	 *
	 * @param store
	 * @param code
	 * @param language
	 * @return ReadableCategory
	 * @throws Exception
	 */
	ReadableCategory getByCode(MerchantStore store, String code, Language language) throws Exception;

	/**
	 * Get a Category by the Search Engine friendly URL slug
	 *
	 * @param merchantStore
	 * @param friendlyUrl
	 * @param language
	 * @return
	 */
	ReadableCategory getCategoryByFriendlyUrl(MerchantStore merchantStore, String friendlyUrl, Language language) throws Exception;

	Category getByCode(String code, MerchantStore store);

	void deleteCategory(Long categoryId, MerchantStore store);

	void deleteCategory(Category category);


	/**
	 * List product options variations for a given category
	 * @param categoryId
	 * @param store
	 * @param language
	 * @return
	 */
	List<ReadableProductVariant> categoryProductVariants(Long categoryId, MerchantStore store, Language language);

	/**
	 * Check if category code already exist
	 * @param store
	 * @param code
	 * @return
	 * @throws Exception
	 */
	boolean existByCode(MerchantStore store, String code);

	/**
	 * Move a Category from a node to another node
	 * @param child
	 * @param parent
	 * @param store
	 */
	void move(Long child, Long parent, MerchantStore store);

	/**
	 * Set category visible or not
	 * @param category
	 * @param store
	 */
	void setVisible(PersistableCategory category, MerchantStore store);
	
	
	/**
	 * List category by product
	 * @param store
	 * @param product
	 * @return
	 */
	ReadableCategoryList listByProduct(MerchantStore store, Long product, Language language);
}



```
