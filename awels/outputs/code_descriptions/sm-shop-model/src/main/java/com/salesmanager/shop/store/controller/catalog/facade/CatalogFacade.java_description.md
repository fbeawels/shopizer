# CatalogFacade.java

## Review

## 1. Summary  

**Purpose**  
`CatalogFacade` is a *facade* interface that exposes CRUD‑like operations for the **catalog** domain objects of an e‑commerce system. It is intentionally thin – the real work is performed by an implementation class (not shown) that talks to the data‑access layer, the business‑logic layer, or external services.  

**Key Components**  

| Component | Role |
|-----------|------|
| `PersistableCatalog` / `PersistableCatalogCategoryEntry` | DTOs that carry data required to *create* or *update* a catalog or an entry. |
| `ReadableCatalog` / `ReadableCatalogCategoryEntry` | DTOs that represent the read‑only view of a catalog or entry. |
| `ReadableEntityList<T>` | A generic wrapper that contains a page of entities together with paging metadata. |
| `MerchantStore` | Contextual information about the store that owns the catalog. |
| `Language` | Locale used for i18n lookups. |
| `Catalog` | The core JPA / domain entity – only used by the `getCatalog(String, MerchantStore)` overload that returns the *raw* entity. |

The interface itself follows the **Facade** design pattern: it offers a clean, service‑like API that hides the complexity of the underlying layers.

**Framework / Library Use**  
The code relies on standard Java (Java 8+ for `Optional`), but it is agnostic to any particular framework (e.g., Spring, Hibernate). The real implementation would likely use JPA/Hibernate, Spring Data, or similar.



## 2. Detailed Description  

### Execution Flow  

1. **Client Interaction** – A controller or service layer calls a method such as `saveCatalog` or `getListCatalogs`.  
2. **Facade Delegation** – The facade implementation delegates to the underlying services/repositories, passing along the `MerchantStore` and `Language` for context.  
3. **Persistence / Business Logic** – The implementation performs validation, resolves references, persists the entity, and returns a DTO (`ReadableCatalog`, `ReadableCatalogCategoryEntry`, or a list).  
4. **Return** – The facade returns a DTO or a primitive flag (`boolean`) to the caller.

No explicit cleanup logic is required at the facade level; any resource management is handled by the implementation (e.g., transaction boundaries, connection pooling).

### Assumptions & Constraints  

| Area | Assumption | Constraint |
|------|------------|------------|
| **Uniqueness** | `uniqueCatalog` expects `code` to be globally unique per store. | Caller must ensure `code` is non‑null and trimmed. |
| **Language** | Methods that take a `Language` assume that the store has a default language and that the code is available in that language. | If the language is `null`, the implementation must fall back or throw an exception. |
| **Pagination** | `page` and `count` are zero‑based and positive integers. | No validation is shown in the interface; implementation must guard against negative values. |
| **Entry Removal** | `removeCatalogEntry` takes both `catalogId` and `catalogEntryId` implying a *relationship* constraint. | The implementation must ensure the entry belongs to the catalog. |
| **Overloaded Getters** | Two `getCatalog(String, MerchantStore, Language)` and `getCatalog(String, MerchantStore)` overloads are provided. | The language‑agnostic overload is likely for backward compatibility; the implementation must decide which locale to use. |

### Architecture & Design Choices  

* **Separation of Concerns** – DTOs are split into *persistable* and *readable* variants, making the contract clear.  
* **Optional Filters** – `Optional<String>` is used for search parameters, which is idiomatic Java 8+ for *absence* of a filter.  
* **Generic Paging** – `ReadableEntityList<T>` abstracts paging and can be reused across domains.  
* **Minimal Exceptions** – The interface does not declare any checked exceptions; it is expected that implementations will throw unchecked runtime exceptions or return null/Optional if an entity is not found.  

These choices favor simplicity but place more responsibility on the implementation to enforce validation and exception handling.



## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `ReadableCatalog saveCatalog(PersistableCatalog catalog, MerchantStore store, Language language)` | Persist a new catalog. | `PersistableCatalog` (data), `MerchantStore`, `Language`. | `ReadableCatalog` (saved catalog). | Creates a database row; may throw runtime exceptions on validation failure. |
| `void updateCatalog(Long catalogId, PersistableCatalog catalog, MerchantStore store, Language language)` | Update an existing catalog identified by `catalogId`. | `Long` id, `PersistableCatalog`, `MerchantStore`, `Language`. | `void`. | Mutates the database row; may throw if catalog not found. |
| `void deleteCatalog(Long catalogId, MerchantStore store, Language language)` | Delete the catalog identified by `catalogId`. | `Long` id, `MerchantStore`, `Language`. | `void`. | Removes the database row; cascades to entries if configured. |
| `boolean uniqueCatalog(String code, MerchantStore store)` | Check if a catalog code is unique within the store. | `String` code, `MerchantStore`. | `boolean`. | No side effects. |
| `ReadableCatalog getCatalog(String code, MerchantStore store, Language language)` | Retrieve a catalog by its code and language. | `String` code, `MerchantStore`, `Language`. | `ReadableCatalog`. | No side effects. |
| `Catalog getCatalog(String code, MerchantStore store)` | Retrieve the *raw* `Catalog` entity (no language). | `String` code, `MerchantStore`. | `Catalog`. | No side effects. |
| `ReadableCatalog getCatalog(Long id, MerchantStore store, Language language)` | Retrieve a catalog by its ID. | `Long` id, `MerchantStore`, `Language`. | `ReadableCatalog`. | No side effects. |
| `ReadableEntityList<ReadableCatalog> getListCatalogs(Optional<String> code, MerchantStore store, Language language, int page, int count)` | List catalogs with optional code filter, paging. | `Optional<String> code`, `MerchantStore`, `Language`, `int page`, `int count`. | `ReadableEntityList<ReadableCatalog>`. | No side effects. |
| `ReadableEntityList<ReadableCatalogCategoryEntry> listCatalogEntry(Optional<String> product, Long catalogId, MerchantStore store, Language language, int page, int count)` | List entries within a catalog, optional product filter, paging. | `Optional<String> product`, `Long catalogId`, `MerchantStore`, `Language`, `int page`, `int count`. | `ReadableEntityList<ReadableCatalogCategoryEntry>`. | No side effects. |
| `ReadableCatalogCategoryEntry getCatalogEntry(Long id, MerchantStore store, Language language)` | Retrieve a single catalog entry by ID. | `Long` id, `MerchantStore`, `Language`. | `ReadableCatalogCategoryEntry`. | No side effects. |
| `ReadableCatalogCategoryEntry addCatalogEntry(PersistableCatalogCategoryEntry entry, MerchantStore store, Language language)` | Persist a new catalog entry. | `PersistableCatalogCategoryEntry`, `MerchantStore`, `Language`. | `ReadableCatalogCategoryEntry`. | Creates a database row. |
| `void removeCatalogEntry(Long catalogId, Long catalogEntryId, MerchantStore store, Language language)` | Remove an entry from a catalog. | `Long` catalogId, `Long` catalogEntryId, `MerchantStore`, `Language`. | `void`. | Deletes the association/row. |

**Reusable / Utility Methods** – None are defined; all methods are domain‑specific. The interface could benefit from generic helper methods (e.g., `exists`, `count`) but those can be added later.



## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `com.salesmanager.core.model.catalog.catalog.Catalog` | Domain | JPA entity (likely). |
| `com.salesmanager.core.model.merchant.MerchantStore` | Domain | Represents a merchant context. |
| `com.salesmanager.core.model.reference.language.Language` | Domain | Locale information. |
| `java.util.Optional` | Standard | Used for optional filters. |
| `ReadableEntityList` | Custom | Generic paging wrapper. |

All dependencies are either part of the project's own codebase or standard Java. No external third‑party libraries are referenced directly in the interface. The concrete implementation, however, would almost certainly use Spring Data, Hibernate, or similar frameworks.



## 5. Additional Notes & Recommendations  

### Edge Cases Not Handled in the Interface  

| Scenario | Issue | Suggested Fix |
|----------|-------|---------------|
| `catalogId` or `id` is `null` | The interface accepts primitive `Long` but callers might pass `null`. | Add input validation in implementation; optionally overload with `Optional<Long>`. |
| Duplicate `code` during `saveCatalog` | `uniqueCatalog` must be called beforehand, but a race condition could still happen. | Enforce a unique constraint at the database level; catch `DataIntegrityViolationException`. |
| Missing `Language` | Methods that take `Language` assume it is non‑null; `null` could lead to NPEs. | Declare `@NonNull` (if using Lombok) or perform a null check and throw an `IllegalArgumentException`. |
| Paging parameters out of bounds (`page < 0`, `count <= 0`) | Could result in an empty list or exception from the repository layer. | Validate and throw `IllegalArgumentException`. |
| `removeCatalogEntry` called with mismatched `catalogId` and `catalogEntryId` | Implementation must verify the relationship. | Perform an ownership check and throw a custom `CatalogEntryNotFoundException` if the entry does not belong to the catalog. |

### Design Improvements  

1. **Consistent Use of Optional** – Either use `Optional` for all “may‑be‑absent” parameters or drop it entirely. Mixing it with non‑optional types can confuse callers.  
2. **Unified Return Types** – Consider returning `Optional<ReadableCatalog>` for methods that may not find an entity (e.g., `getCatalog(String, ...)`). This eliminates the need to handle `null`.  
3. **Exception Strategy** – The interface should document expected exceptions (e.g., `CatalogNotFoundException`, `DuplicateCatalogCodeException`) instead of relying on unchecked runtime exceptions.  
4. **Method Naming** – `removeCatalogEntry` could be renamed to `deleteCatalogEntry` to be consistent with `deleteCatalog`.  
5. **Overloaded `getCatalog`** – The two overloads that differ only by language may be confusing. If language‑agnostic access is needed, consider a default language or a separate `CatalogRepository` interface.  
6. **Generic Paging Utility** – `ReadableEntityList` could include total count, page number, and page size for better client navigation.  

### Future Enhancements  

* **Batch Operations** – Methods for bulk add/update/delete of catalog entries.  
* **Search / Filter DSL** – Replace `Optional<String>` filters with a richer search specification (e.g., `CatalogSearchCriteria`).  
* **Event Publishing** – Fire domain events (e.g., `CatalogCreatedEvent`) upon successful writes for eventual consistency or auditing.  
* **Caching** – Add a caching layer for read‑heavy operations (`getCatalog`, `listCatalogEntry`).  

---

**Verdict** – The interface is clean and expressive, clearly delineating CRUD responsibilities for catalogs and their entries. With minor adjustments around null safety, optional handling, and exception contracts, it would provide a robust contract for implementations and downstream consumers.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.catalog.facade;

import com.salesmanager.core.model.catalog.catalog.Catalog;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.catalog.*;
import com.salesmanager.shop.model.entity.ReadableEntityList;

import java.util.Optional;

public interface CatalogFacade {

    ReadableCatalog saveCatalog(PersistableCatalog catalog, MerchantStore store, Language language);

    void updateCatalog(Long catalogId, PersistableCatalog catalog, MerchantStore store, Language language);

    void deleteCatalog(Long catalogId, MerchantStore store, Language language);

    boolean uniqueCatalog(String code, MerchantStore store);

    ReadableCatalog getCatalog(String code, MerchantStore store, Language language);

    Catalog getCatalog(String code, MerchantStore store);

    ReadableCatalog getCatalog(Long id, MerchantStore store, Language language);

    ReadableEntityList<ReadableCatalog> getListCatalogs(Optional<String> code, MerchantStore store, Language language, int page, int count);

    ReadableEntityList<ReadableCatalogCategoryEntry> listCatalogEntry(Optional<String> product, Long catalogId, MerchantStore store, Language language, int page, int count);

    ReadableCatalogCategoryEntry getCatalogEntry(Long id, MerchantStore store, Language language);

    ReadableCatalogCategoryEntry addCatalogEntry(PersistableCatalogCategoryEntry entry, MerchantStore store, Language language);

    void removeCatalogEntry(Long catalogId, Long catalogEntryId, MerchantStore store, Language language);

}



```
