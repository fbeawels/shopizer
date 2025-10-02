# MerchantStoreServiceImpl.java

## Review

## 1. Summary  
**Purpose** – `MerchantStoreServiceImpl` is the main business‑logic façade for handling *MerchantStore* entities (stores, retailers, and their groups). It provides CRUD, search, and relationship‑navigation operations that sit on top of Spring Data JPA repositories.  

**Key components**  
| Component | Role | Notes |
|-----------|------|-------|
| `MerchantStoreServiceImpl` | Service implementation | Extends a generic `SalesManagerEntityServiceImpl` to inherit basic CRUD and implements `MerchantStoreService`. |
| `MerchantRepository` | Spring Data JPA repository | Handles simple queries such as `findByCode`, `existsByCode`, and various custom `@Query` methods. |
| `PageableMerchantRepository` | Repository that supports paginated custom queries | Provides `Page<MerchantStore>` results for list operations. |
| `ProductTypeService` | (Injected but unused in the provided snippet) | Likely used elsewhere in the full class. |
| `Pageable` / `PageRequest` | Spring pagination | Enables page‑aware queries. |
| `Validate` (from *jsoup*) | Argument validation | Only used in `getParent`. |

**Design patterns & frameworks**  
- **Repository pattern** (Spring Data JPA).  
- **Service layer** (business logic separation).  
- **Generic base service** (`SalesManagerEntityServiceImpl`).  
- Optional usage of **Optional** (although not fully leveraged).  
- Spring’s **dependency injection** and **transactional** handling (inherited).  

---

## 2. Detailed Description  

### Flow of execution
1. **Construction** – The service is created by Spring (`@Service`).  
   - `MerchantRepository` is injected via constructor and stored in a private field.  
   - `PageableMerchantRepository` is autowired.  
   - `ProductTypeService` is also injected (unused in this snippet).  

2. **CRUD / Queries**  
   - **`saveOrUpdate`** – Delegates to the generic `save` method.  
   - **`getByCode`** – Retrieves a store by its unique code.  
   - **`existByCode`** – Checks existence by code.  
   - **`getByCriteria`** – Delegates to a custom repository method returning a list wrapper.  
   - **Paginated listing** – `listChildren`, `listAll`, `listAllRetailers`, `listByGroup` use `PageableMerchantRepository`.  
   - **Relationship navigation** – `getParent` finds the root store for a given code.  
   - **Group checks** – `isStoreInGroup` determines if a store belongs to a group.  

3. **Error handling** – All public methods declare `throws ServiceException`.  
   - The service relies on the repository to return `null` for missing data; it translates missing data into `ServiceException` or relies on Spring to propagate.  
   - `Validate.notNull` is used only in `getParent`.  

4. **Cleanup** – No explicit cleanup; relies on Spring container.

### Assumptions & constraints  
- `MerchantStore` objects are persisted via JPA and contain `id`, `code`, `name`, `isRetailer`, and an optional `parent` relationship.  
- Repository methods return `null` for missing entities; the service sometimes converts that to exceptions.  
- The service is stateless and thread‑safe.  
- Pagination parameters (`page`, `count`) are zero‑based (`PageRequest.of(page, count)`).  

### Architecture  
A conventional layered architecture: **Controller → Facade/Facade‑like layer (not shown) → Service → Repository → Database**.  
Caching annotations (`@Cacheable`, `@CacheEvict`) are commented out, suggesting caching is handled at the facade layer instead.  

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Output | Side‑Effects |
|--------|---------|--------|--------|--------------|
| `saveOrUpdate(MerchantStore store)` | Persist or update a store. | `store` | void | Delegates to `super.save(store)` which persists the entity. |
| `getByCode(String code)` | Retrieve a store by its unique code. | `code` | `MerchantStore` or `null` | None. |
| `existByCode(String code)` | Boolean existence check. | `code` | `boolean` | None. |
| `getByCriteria(MerchantStoreCriteria criteria)` | Custom criteria‑based list. | `criteria` | `GenericEntityList<MerchantStore>` | None. |
| `listChildren(String code, int page, int count)` | Get paginated children of a store. | `code`, `page`, `count` | `Page<MerchantStore>` | None. |
| `listAll(Optional<String> storeName, int page, int count)` | Paginated list of all stores, optionally filtered by name. | `storeName`, `page`, `count` | `Page<MerchantStore>` | None. |
| `findAllStoreCodeNameEmail()` | Fetch minimal data for all stores (code, name, email). | None | `List<MerchantStore>` | None. |
| `listAllRetailers(Optional<String> storeName, int page, int count)` | Paginated list of all retailer stores. | `storeName`, `page`, `count` | `Page<MerchantStore>` | None. |
| `findAllStoreNames()` | List of all store names. | None | `List<MerchantStore>` | None. |
| `getParent(String code)` | Resolve the parent store for a given code, handling retailers as roots. | `code` | `MerchantStore` | Throws `ServiceException` if not found. |
| `findAllStoreNames(String code)` | List of store names filtered by a given code. | `code` | `List<MerchantStore>` | None. |
| `listByGroup(Optional<String> storeName, String code, int page, int count)` | Get paginated list of stores in the same group as `code`, optionally filtered by name. | `storeName`, `code`, `page`, `count` | `Page<MerchantStore>` | Throws `ServiceException` if code not found. |
| `isStoreInGroup(String code)` | Check whether a store has any children (is a parent). | `code` | `boolean` | None. |

### Utility / Reusable methods  
- The repeated pattern of extracting a `String` from an `Optional<String>` is duplicated; a private helper could reduce duplication.  
- The use of `Validate.notNull` could be moved to all public methods for consistent argument checking.  

---

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| **Spring Framework** (`@Service`, `@Autowired`, `Pageable`, `PageRequest`) | Third‑party | Dependency injection, data paging, service stereotype. |
| **Spring Data JPA** (`Page`, repository interfaces) | Third‑party | ORM and repository abstractions. |
| **Jsoup** (`Validate`) | Third‑party | Simple validation helper. |
| **SalesManagerCore** (project-specific) | Project | `SalesManagerEntityServiceImpl`, `ServiceException`, model classes, custom repositories. |

All dependencies are either standard Spring ecosystem components or project‑specific code. No platform‑specific APIs are visible.

---

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns – service logic is distinct from persistence.  
* Consistent use of Spring Data’s paging utilities.  
* Generic base service reduces boilerplate CRUD code.  
* Exception handling via `ServiceException` keeps API contracts explicit.  

### Potential Issues & Edge Cases  
1. **Null Handling** –  
   * `getByCode` may return `null`. Methods like `listByGroup`, `isStoreInGroup` call `store.getId()` without null checks, risking `NullPointerException`.  
   * The `Optional.ofNullable(store.getId())` pattern followed by `id.get()` defeats the purpose of `Optional`. It can throw `NoSuchElementException` if `id` is empty.  
2. **Redundant Field / Injection** –  
   * `merchantRepository` is both a constructor argument and a field. The field is also `private` but reassigned; this redundancy is confusing. Prefer constructor injection with `final` fields.  
3. **Inconsistent Validation** –  
   * Only `getParent` uses `Validate.notNull`. All public methods should validate required parameters to avoid silent failures.  
4. **Code Duplication** –  
   * Repeated extraction of `String` from `Optional<String>` in several methods. A private helper (`private static String getString(Optional<String> opt)`) would simplify the code.  
5. **Caching** –  
   * Commented `@Cacheable`/`@CacheEvict` annotations suggest caching is not handled here. Ensure the facade layer implements proper cache eviction for `saveOrUpdate`.  
6. **Naming** –  
   * Method names like `listAllRetailers` and `listAll` are ambiguous; consider more descriptive names (e.g., `listRetailersPaginated`).  
7. **Transactional Boundaries** –  
   * The base service may mark methods as transactional; however, read‑only queries can be annotated with `@Transactional(readOnly = true)` for potential performance benefits.  

### Future Enhancements  
* **Use Optional in repository return types** – Change repository methods to return `Optional<MerchantStore>` to avoid `null` handling.  
* **Add @Transactional annotations** – Explicitly mark methods as read‑only or read‑write where appropriate.  
* **Introduce a `StoreNotFoundException`** – More specific than generic `ServiceException`.  
* **Cache at the service layer** – Re‑enable caching annotations or implement a caching interceptor if performance requires.  
* **Unit tests** – Mock the repositories and verify behavior for edge cases (missing store, retailer detection).  
* **Refactor repeated logic** – Extract common pagination and optional extraction helpers.  

Overall, the implementation follows conventional Spring practices and provides the core business operations required for merchant store management. Addressing the null‑safety and duplication concerns will improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.merchant;

import java.util.List;
import java.util.Optional;

import javax.inject.Inject;

import org.jsoup.helper.Validate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.merchant.MerchantRepository;
import com.salesmanager.core.business.repositories.merchant.PageableMerchantRepository;
import com.salesmanager.core.business.services.catalog.product.type.ProductTypeService;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.common.GenericEntityList;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.merchant.MerchantStoreCriteria;

@Service("merchantService")
public class MerchantStoreServiceImpl extends SalesManagerEntityServiceImpl<Integer, MerchantStore>
		implements MerchantStoreService {

	@Inject
	protected ProductTypeService productTypeService;

	@Autowired
	private PageableMerchantRepository pageableMerchantRepository;

	private MerchantRepository merchantRepository;

	@Inject
	public MerchantStoreServiceImpl(MerchantRepository merchantRepository) {
		super(merchantRepository);
		this.merchantRepository = merchantRepository;
	}

	@Override
	//@CacheEvict(value="store", key="#store.code")
	public void saveOrUpdate(MerchantStore store) throws ServiceException {
		super.save(store);
	}

	@Override
	/**
	 * cache moved in facades
	 */
	//@Cacheable(value = "store")
	public MerchantStore getByCode(String code) throws ServiceException {
		return merchantRepository.findByCode(code);
	}

	@Override
	public boolean existByCode(String code) {
		return merchantRepository.existsByCode(code);
	}

	@Override
	public GenericEntityList<MerchantStore> getByCriteria(MerchantStoreCriteria criteria) throws ServiceException {
		return merchantRepository.listByCriteria(criteria);
	}

	@Override
	public Page<MerchantStore> listChildren(String code, int page, int count) throws ServiceException {
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableMerchantRepository.listByStore(code, pageRequest);
	}

	@Override
	public Page<MerchantStore> listAll(Optional<String> storeName, int page, int count) throws ServiceException {
		String store = null;
		if (storeName != null && storeName.isPresent()) {
			store = storeName.get();
		}
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableMerchantRepository.listAll(store, pageRequest);

	}

	@Override
	public List<MerchantStore> findAllStoreCodeNameEmail() throws ServiceException {
		return merchantRepository.findAllStoreCodeNameEmail();
	}

	@Override
	public Page<MerchantStore> listAllRetailers(Optional<String> storeName, int page, int count)
			throws ServiceException {
		String store = null;
		if (storeName != null && storeName.isPresent()) {
			store = storeName.get();
		}
		Pageable pageRequest = PageRequest.of(page, count);
		return pageableMerchantRepository.listAllRetailers(store, pageRequest);

	}

	@Override
	public List<MerchantStore> findAllStoreNames() throws ServiceException {
		return merchantRepository.findAllStoreNames();
	}

	@Override
	public MerchantStore getParent(String code) throws ServiceException {
		Validate.notNull(code, "MerchantStore code cannot be null");

		
		//get it
		MerchantStore storeModel = this.getByCode(code);
		
		if(storeModel == null) {
			throw new ServiceException("Store with code [" + code + "] is not found");
		}
		
		if(storeModel.isRetailer() != null && storeModel.isRetailer() && storeModel.getParent() == null) {
			return storeModel;
		}
		
		if(storeModel.getParent() == null) {
			return storeModel;
		}
	
		return merchantRepository.getById(storeModel.getParent().getId());
	}


	@Override
	public List<MerchantStore> findAllStoreNames(String code) throws ServiceException {
		return merchantRepository.findAllStoreNames(code);
	}

	/**
	 * Store might be alone (known as retailer)
	 * A retailer can have multiple child attached
	 * 
	 * This method from a store code is able to retrieve parent and childs.
	 * Method can also filter on storeName
	 */
	@Override
	public Page<MerchantStore> listByGroup(Optional<String> storeName, String code, int page, int count) throws ServiceException {
		
		String name = null;
		if (storeName != null && storeName.isPresent()) {
			name = storeName.get();
		}

		
		MerchantStore store = getByCode(code);//if exist
		Optional<Integer> id = Optional.ofNullable(store.getId());

		
		Pageable pageRequest = PageRequest.of(page, count);


		return pageableMerchantRepository.listByGroup(code, id.get(), name, pageRequest);
		
		
	}

	@Override
	public boolean isStoreInGroup(String code) throws ServiceException{
		
		MerchantStore store = getByCode(code);//if exist
		Optional<Integer> id = Optional.ofNullable(store.getId());
		
		List<MerchantStore> stores = merchantRepository.listByGroup(code, id.get());
		
		
		return stores.size() > 0;
	}


}



```
