# StoreFacade.java

## Review

## 1. Summary  
The `StoreFacade` interface is a thin service layer that mediates between the shop‑layer (controllers, API endpoints) and the core business logic in `sm-core`.  
Its responsibilities are:  

* **MerchantStore CRUD** – create, read, update, delete stores.  
* **Brand & Logo handling** – retrieve brand configuration, create a brand, upload/delete logos.  
* **Search & paging** – list stores by various criteria, support pagination.  
* **Language support** – return locale‑aware representations (`ReadableMerchantStore`) and the list of supported languages for a store.  

The interface relies on several domain/value objects (`MerchantStore`, `PersistableMerchantStore`, `ReadableMerchantStore`, `ReadableMerchantStoreList`, `PersistableBrand`, `ReadableBrand`, `Language`, `MerchantStoreCriteria`) and a lightweight file wrapper (`InputContentFile`) for logo uploads. No external frameworks are referenced directly, although the API is designed to be wired by a dependency‑injection container (Spring, CDI, etc.) in the surrounding application.

---

## 2. Detailed Description  

### Core Flow  
1. **Initialization** – Implementations of this facade will typically be injected into controllers.  
2. **Read Operations** – Methods such as `getByCode`, `getFullByCode`, `getByCriteria`, and `findAll` translate incoming parameters (code, language, pagination, criteria) into a service layer call that returns domain or DTO objects.  
3. **Write Operations** – `create`, `update`, `delete`, `createBrand`, `addStoreLogo`, and `deleteLogo` perform mutating actions. They usually trigger validation, persistence, and possibly events.  
4. **Cleanup** – Not applicable to the interface itself; implementations may manage transactions or cleanup resources, e.g. closing file streams.

### Assumptions & Constraints  
* The caller is responsible for providing a valid store code or `HttpServletRequest` that contains the code (likely from a request header or path variable).  
* Language objects represent a locale; the facade assumes that a matching locale exists for a store.  
* The `MerchantStore` domain object is mutable and may contain more data than the read‑only DTOs.  
* All operations are expected to be **atomic**; the implementing class should handle transaction boundaries.  
* No explicit exception contracts are declared – implementations may throw unchecked exceptions or wrap checked ones.

### Architectural Choices  
* **Facade Pattern** – Exposes a simplified API to the presentation layer, hiding the complexity of underlying services.  
* **DTO vs Domain Separation** – Read‑only representations (`ReadableMerchantStore`, `ReadableBrand`) keep the UI decoupled from internal domain objects.  
* **Language‑Aware Retrieval** – Methods overload to accept either a `String` code or a `Language` object, facilitating both generic and locale‑specific lookups.  
* **Paging Parameters** – `page` and `count` are used for server‑side pagination; however, a more expressive `PageRequest` type might be clearer.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Output | Side‑Effects | Notes |
|--------|---------|--------|--------|--------------|-------|
| `MerchantStore getByCode(HttpServletRequest request)` | Derives store code from request (e.g., header, attribute) and retrieves the store domain object. | `HttpServletRequest` | `MerchantStore` | None | Unclear mapping logic; might expose implementation details. |
| `MerchantStore get(String code)` | Retrieve store by code. | `String` | `MerchantStore` | None | Duplicate of `getByCode(String)` – consider removing one. |
| `MerchantStore getByCode(String code)` | Retrieve store by code. | `String` | `MerchantStore` | None | Same as `get(String)`. |
| `List<Language> supportedLanguages(MerchantStore store)` | List locales supported by a store. | `MerchantStore` | `List<Language>` | None | Useful for UI language selector. |
| `ReadableMerchantStore getByCode(String code, String lang)` | Get a locale‑aware DTO by store code and language code. | `String`, `String` | `ReadableMerchantStore` | None | Requires language lookup internally. |
| `ReadableMerchantStore getFullByCode(String code, String lang)` | Get a full‑detail DTO by code/language. | `String`, `String` | `ReadableMerchantStore` | None | Could differentiate from `getByCode` by including additional data. |
| `ReadableMerchantStoreList findAll(MerchantStoreCriteria criteria, Language language, int page, int count)` | List stores matching criteria with pagination. | `MerchantStoreCriteria`, `Language`, `int`, `int` | `ReadableMerchantStoreList` | None | Pagination parameters imply offset/limit semantics. |
| `ReadableMerchantStoreList getChildStores(Language language, String code, int start, int count)` | Retrieve child stores of a parent store. | `Language`, `String`, `int`, `int` | `ReadableMerchantStoreList` | None | `start` is offset, `count` is limit. |
| `ReadableMerchantStore getByCode(String code, Language lang)` | Locale‑aware retrieval using `Language` object. | `String`, `Language` | `ReadableMerchantStore` | None | Overloaded version of `getByCode(String, String)`. |
| `ReadableMerchantStore getFullByCode(String code, Language language)` | Full detail retrieval using `Language` object. | `String`, `Language` | `ReadableMerchantStore` | None | Overloaded version of `getFullByCode(String, String)`. |
| `boolean existByCode(String code)` | Check if a store exists. | `String` | `boolean` | None | Could be named `existsByCode`. |
| `ReadableMerchantStoreList getByCriteria(MerchantStoreCriteria criteria, Language lang)` | Retrieve stores by criteria (no pagination). | `MerchantStoreCriteria`, `Language` | `ReadableMerchantStoreList` | None | Similar to `findAll` but without paging. |
| `void create(PersistableMerchantStore store)` | Persist a new store. | `PersistableMerchantStore` | void | Persisted to DB, may trigger events. | Previously returned `ReadableMerchantStore`. |
| `void update(PersistableMerchantStore store)` | Update an existing store. | `PersistableMerchantStore` | void | Updated DB row. | Previously returned `ReadableMerchantStore`. |
| `void delete(String code)` | Remove store by code. | `String` | void | Deleted from DB. | May need cascade cleanup. |
| `ReadableBrand getBrand(String code)` | Retrieve brand configuration for a store. | `String` | `ReadableBrand` | None | Includes logo, social links, etc. |
| `void createBrand(String merchantStoreCode, PersistableBrand brand)` | Create brand data for a store. | `String`, `PersistableBrand` | void | Persisted brand. | No return value; client must fetch afterwards. |
| `void deleteLogo(String code)` | Delete the logo of a store. | `String` | void | File removed from storage. | Might also clear DB reference. |
| `void addStoreLogo(String code, InputContentFile cmsContentImage)` | Upload or replace a store’s logo. | `String`, `InputContentFile` | void | Stores file, updates DB. | Needs validation of image type/size. |
| `List<ReadableMerchantStore> getMerchantStoreNames(MerchantStoreCriteria criteria)` | Return minimal store info (id, code, name). | `MerchantStoreCriteria` | `List<ReadableMerchantStore>` | None | Useful for dropdowns. |

**Reusable/Utility Methods**  
None are defined at this level; however, the interface promotes reuse by exposing domain‑neutral DTOs (`ReadableMerchantStore`, `ReadableBrand`).

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `MerchantStore`, `PersistableMerchantStore`, `ReadableMerchantStore`, `ReadableMerchantStoreList` | Domain/DTO | Core store representations. |
| `PersistableBrand`, `ReadableBrand` | Domain/DTO | Brand information. |
| `Language` | Value Object | Locale handling. |
| `MerchantStoreCriteria` | Criteria | Search filters. |
| `InputContentFile` | File wrapper | Logo upload abstraction. |
| `HttpServletRequest` | Servlet API | Extraction of store code from web context. |

All dependencies are **domain/value objects** defined within the `com.salesmanager` namespace, except for `HttpServletRequest` which comes from the Java Servlet API (standard library). No third‑party libraries are referenced directly by the interface; however, concrete implementations will likely rely on Spring, JPA, or other persistence frameworks.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation** between domain objects and read‑only DTOs keeps the UI decoupled from persistence logic.  
* **Language awareness** baked into many methods facilitates internationalization.  
* **Facade pattern** keeps the controller layer simple and focused on request handling.

### Potential Issues / Edge Cases  
1. **Redundant Method Signatures** – `get(String)` vs `getByCode(String)` and overloaded language parameters can lead to confusion and maintenance overhead.  
2. **Missing Error Handling** – The interface does not declare any checked exceptions. Implementations may throw unchecked exceptions; consider using a custom `StoreNotFoundException` or `InvalidInputException` for clarity.  
3. **HTTP Request Coupling** – `getByCode(HttpServletRequest request)` introduces a web‑specific dependency in a domain‑layer interface; this breaks separation of concerns. It would be cleaner to extract the store code in the controller and pass it to the facade.  
4. **Naming Consistency** – Methods like `existByCode` could be renamed to `existsByCode` to align with Java naming conventions.  
5. **Return Types on Mutations** – The create/update methods currently return `void`. Returning the persisted DTO (as the commented‑out signatures suggested) can improve usability and allow callers to immediately use the new data without an additional fetch.  
6. **Pagination Parameters** – The use of `start`/`count` or `page`/`count` can be inconsistent. Introducing a unified `PageRequest` type would simplify method signatures.  
7. **Logo Handling** – `addStoreLogo` and `deleteLogo` assume a storage backend; error handling for file I/O should be documented.  
8. **Transactional Safety** – Mutating operations should be wrapped in a transaction; the interface cannot enforce this, so documentation or an aspect should be added.

### Future Enhancements  
* **Optional Return Types** – Methods like `getByCode` could return `Optional<MerchantStore>` to explicitly model absence.  
* **Bulk Operations** – Support for bulk create/update/delete of stores or brands.  
* **Event Publishing** – Emit domain events after create/update/delete to allow asynchronous processing (e.g., cache refresh, audit logging).  
* **Extended Brand API** – Methods to update or delete brand data and social links.  
* **Audit Fields** – Include created/modified timestamps and user identifiers in DTOs.  
* **Security & Authorization** – Interface could accept a security context or user identifier to enforce per‑store permissions.

---

**Overall Verdict**  
The `StoreFacade` interface provides a solid, high‑level contract for store‑related operations in the shop layer. Minor refactoring (removing redundancies, clarifying method contracts, decoupling from `HttpServletRequest`) would improve clarity and maintainability. The design is extensible, but future iterations should consider the noted edge cases and potential enhancements.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.store.facade;

import java.util.List;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.merchant.MerchantStoreCriteria;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.store.PersistableBrand;
import com.salesmanager.shop.model.store.PersistableMerchantStore;
import com.salesmanager.shop.model.store.ReadableBrand;
import com.salesmanager.shop.model.store.ReadableMerchantStore;
import com.salesmanager.shop.model.store.ReadableMerchantStoreList;

/**
 * Layer between shop controllers, services and API with sm-core
 * 
 * @author carlsamson
 *
 */
public interface StoreFacade {

	/**
	 * Find MerchantStore model from store code
	 * 
	 * @param code
	 * @return
	 * @throws Exception
	 */
	MerchantStore getByCode(HttpServletRequest request);

	MerchantStore get(String code);

	MerchantStore getByCode(String code);
	
	List<Language> supportedLanguages(MerchantStore store);

	ReadableMerchantStore getByCode(String code, String lang);

	ReadableMerchantStore getFullByCode(String code, String lang);

	ReadableMerchantStoreList findAll(MerchantStoreCriteria criteria, Language language, int page, int count);

	/**
	 * List child stores
	 * 
	 * @param code
	 * @return
	 */
	ReadableMerchantStoreList getChildStores(Language language, String code, int start, int count);

	ReadableMerchantStore getByCode(String code, Language lang);

	ReadableMerchantStore getFullByCode(String code, Language language);

	boolean existByCode(String code);

	/**
	 * List MerchantStore using various criterias
	 * 
	 * @param criteria
	 * @param lang
	 * @return
	 * @throws Exception
	 */
	ReadableMerchantStoreList getByCriteria(MerchantStoreCriteria criteria, Language lang);

	/**
	 * Creates a brand new MerchantStore
	 * 
	 * @param store
	 * @throws Exception
	 */
	//ReadableMerchantStore create(PersistableMerchantStore store);
	void create(PersistableMerchantStore store);

	/**
	 * Updates an existing store
	 * 
	 * @param store
	 * @throws Exception
	 */
	//ReadableMerchantStore update(PersistableMerchantStore store);
	void update(PersistableMerchantStore store);

	/**
	 * Deletes a MerchantStore based on store code
	 * 
	 * @param code
	 */
	void delete(String code);

	/**
	 * Get Logo, social networks and other brand configurations
	 * 
	 * @param code
	 * @return
	 */
	ReadableBrand getBrand(String code);

	/**
	 * Create store brand
	 * 
	 * @param merchantStoreCode
	 * @param brand
	 */
	void createBrand(String merchantStoreCode, PersistableBrand brand);

	/**
	 * Delete store logo
	 */
	void deleteLogo(String code);

	/**
	 * Add MerchantStore logo
	 * 
	 * @param code
	 * @param cmsContentImage
	 */
	void addStoreLogo(String code, InputContentFile cmsContentImage);

	/**
	 * Returns store id, code and name only
	 * 
	 * @return
	 */
	List<ReadableMerchantStore> getMerchantStoreNames(MerchantStoreCriteria criteria);

}



```
