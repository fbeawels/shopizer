# PageableMerchantRepository.java

## Review

## 1. Summary  

The file defines a **Spring Data JPA repository** (`PageableMerchantRepository`) that provides paging‑and‑sorting capabilities for the `MerchantStore` entity.  
All methods are custom query definitions (JPQL or native SQL) annotated with `@Query`. The repository focuses on **searching merchants** based on:

* Parent store code  
* Store name (partial match)  
* Retailer flag  
* Child/descendant stores  
* Group‑level lookups

Key aspects  

| Feature | Description |
|---------|-------------|
| **Paging & Sorting** | Uses `Pageable` to retrieve results in pages. |
| **JPQL fetch joins** | Many queries use `left join fetch` to eagerly load associated entities (`country`, `currency`, `zone`, `languages`, `parent`). |
| **Native SQL** | The `listByGroup` method falls back to a raw SQL query to leverage database‑specific schema placeholders. |
| **Count queries** | Each JPQL query provides a separate `countQuery` for accurate pagination totals. |

The repository is a plain interface; Spring Data JPA will generate the implementation at runtime.

---

## 2. Detailed Description  

### Core Components  

| Component | Role |
|-----------|------|
| `PageableMerchantRepository` | Spring Data repository interface extending `PagingAndSortingRepository`. |
| `@Query` annotations | Provide custom JPQL/native queries for the required search scenarios. |
| `Page<MerchantStore>` | Return type that encapsulates the result list + pagination metadata. |
| `Pageable` | Parameter used to control page size, sorting, and offset. |

### Execution Flow  

1. **Application startup** – Spring scans the package, detects the interface, and creates a proxy implementation that delegates to the JPA provider.  
2. **Client call** – A service method injects the repository and invokes one of the query methods.  
3. **Query resolution** – Spring uses the JPQL or native SQL defined in `@Query`.  
4. **Pagination** – The `Pageable` argument is applied to the query; Spring automatically adds `LIMIT/OFFSET` (or equivalent) to the JPQL and executes the `countQuery` to compute total pages.  
5. **Result conversion** – JPA maps the rows back to `MerchantStore` entities (with eager fetch joins), wraps them in a `PageImpl`, and returns.  

### Design & Constraints  

* **Eager fetching** – The JPQL queries use `fetch` joins to avoid the N+1 problem when retrieving related entities. However, the `countQuery` does *not* include these joins, which is correct because counts should be performed on the root entity only.  
* **Hard‑coded aliases** – Several queries re‑use the same alias (`mc`) for both `country` and `currency`. In JPQL this will cause a compilation error.  
* **Native SQL** – The `listByGroup` method bypasses JPA’s dialect handling; it relies on `{h-schema}` placeholder, which is specific to Hibernate. This reduces portability.  
* **Parameter placement** – All queries use positional parameters (`?1`, `?2`, …). Spring Data will map them in declaration order, which is clear but less readable than named parameters.  

### Architecture Choices  

* **Repository‑Only Layer** – Keeps business logic in services, delegating all persistence concerns to Spring Data.  
* **Explicit Count Queries** – Necessary for correct pagination when using JPQL with `distinct`.  
* **Eager Loading via JPQL** – Preferred over `@EntityGraph` in this context, though the latter could simplify the code.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `listByStore(String code, Pageable pageable)` | Retrieve all merchant stores that have a parent with the given code. | `code` – parent store code; `pageable` – pagination/sorting | `Page<MerchantStore>` of matched stores. | None (read‑only). |
| `listAll(String storeName, Pageable pageable)` | Retrieve all merchant stores whose name contains `storeName` (case‑sensitive). | `storeName` – optional search term; `pageable` | `Page<MerchantStore>` | None. |
| `listAllRetailers(String storeName, Pageable pageable)` | Retrieve all merchant stores flagged as retailers, optionally filtered by name. | `storeName`, `pageable` | `Page<MerchantStore>` | None. |
| `listChilds(String storeCode, String storeName, Pageable pageable)` | Retrieve child (or same‑level) stores of the store identified by `storeCode`, optionally filtering by name. | `storeCode`, `storeName`, `pageable` | `Page<MerchantStore>` | None. |
| `listByGroup(String storeCode, Integer id, String storeName, Pageable pageable)` | Native SQL query to find stores within a group or by specific ID and name. | `storeCode`, `id`, `storeName`, `pageable` | `Page<MerchantStore>` | None. |

### Reusable / Utility Methods  

None – the repository only contains query methods. If the same query logic were needed elsewhere, a default method or a shared `Specification` could be introduced.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.data.jpa.repository.Query` | Annotation | Spring Data JPA |
| `org.springframework.data.domain.Page` / `Pageable` | Spring Data | Pagination abstraction |
| `org.springframework.data.repository.PagingAndSortingRepository` | Interface | Provides CRUD + paging |
| `com.salesmanager.core.model.merchant.MerchantStore` | Entity | Domain model |
| **Third‑party** |  | Hibernate (implied by `{h-schema}` placeholder) |
| **Platform** |  | Relies on JPA provider (usually Hibernate) and a relational DB that supports the native query syntax. |

No external libraries beyond Spring Data JPA/Hibernate are used.

---

## 5. Additional Notes  

### Issues & Edge Cases  

1. **Duplicate Alias (`mc`)**  
   * In several JPQL queries the alias `mc` is used for both `country` and `currency`. This will cause a `QuerySyntaxException`.  
   * **Fix**: use distinct aliases, e.g., `mc` for `country` and `mu` for `currency`.

2. **Mixed `left join fetch` and `join fetch`**  
   * The `listAllRetailers` count query uses a plain `join` while the data query uses `left join fetch`.  
   * Consistency is not required but can confuse developers reading the code.

3. **Native Query Placeholder**  
   * `{h-schema}` is a Hibernate‑specific placeholder. If the application switches to another JPA provider, the query will fail.  
   * Consider using `@Query(nativeQuery = true)` with a `SchemaName` variable or let JPA generate the schema automatically.

4. **Wildcards in `like` Clauses**  
   * JPQL supports concatenation of literals with parameters, so `like %?1%` is fine.  
   * However, if `storeName` is `null`, the expression `?1 is null` will short‑circuit; the `%?1%` part will still be parsed but never evaluated.

5. **Potential N+1 on Collections**  
   * The `left join fetch m.languages mls` eagerly loads the `languages` collection. If there are many languages per store, this could lead to large result sets.  
   * Consider using `@EntityGraph` or a DTO projection if the full collection is not always needed.

6. **Pagination on `distinct`**  
   * Using `select distinct m` with fetch joins can cause duplicate root entities if the collection is larger than one.  
   * `distinct` in JPQL is applied after fetching, so pagination might not behave as expected. Test with large datasets.

### Potential Enhancements  

* **Method Naming** – Spring Data can derive queries from method names; this would eliminate the need for explicit JPQL in many cases.  
* **Specifications / Criteria API** – Using `JpaSpecificationExecutor` would allow building queries dynamically and combining predicates.  
* **Entity Graphs** – Replace manual fetch joins with `@EntityGraph` annotations to simplify JPQL.  
* **DTO Projections** – For list views, return DTOs instead of full entities to reduce payload and avoid lazy loading issues.  
* **Logging & Monitoring** – Add query logging in the service layer to monitor performance, especially for the native query.  
* **Unit Tests** – Create integration tests with an in‑memory database (H2) to validate query correctness and pagination.

---

**Verdict** – The repository provides the necessary paging queries for merchant stores, but several JPQL alias conflicts and a native‑query dependency need addressing. Once fixed, the interface will be robust, readable, and easier to maintain.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.repositories.merchant;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.PagingAndSortingRepository;

import com.salesmanager.core.model.merchant.MerchantStore;

public interface PageableMerchantRepository extends PagingAndSortingRepository<MerchantStore, Long> {

	/*
	 * List by parent store
	 */
	@Query(value = "select distinct m from MerchantStore m left join fetch m.parent mp left join fetch m.country mc left join fetch m.currency mc left join fetch m.zone mz left join fetch m.defaultLanguage md left join fetch m.languages mls where mp.code = ?1", countQuery = "select count(distinct m) from MerchantStore m join m.parent mp where mp.code = ?1")
	Page<MerchantStore> listByStore(String code, Pageable pageable);

	@Query(value = "select distinct m from MerchantStore m left join fetch m.parent mp left join fetch m.country mc left join fetch m.currency mc left join fetch m.zone mz left join fetch m.defaultLanguage md left join fetch m.languages mls where (?1 is null or m.storename like %?1%)", countQuery = "select count(distinct m) from MerchantStore m where (?1 is null or m.storename like %?1%)")
	Page<MerchantStore> listAll(String storeName, Pageable pageable);

	@Query(value = "select distinct m from MerchantStore m left join fetch m.parent mp "
			+ "left join fetch m.country mc " + "left join fetch m.currency mc left " + "join fetch m.zone mz "
			+ "left join fetch m.defaultLanguage md " + "left join fetch m.languages mls "
			+ "where m.retailer = true and (?1 is null or m.storename like %?1%)", countQuery = "select count(distinct m) from MerchantStore m join m.parent "
					+ "where m.retailer = true and (?1 is null or m.storename like %?1%)")
	Page<MerchantStore> listAllRetailers(String storeName, Pageable pageable);

	@Query(value = "select distinct m from MerchantStore m left join m.parent mp " + "left join fetch m.country pc "
			+ "left join fetch m.currency pcu " + "left join fetch m.languages pl " + "left join fetch m.zone pz "
			+ "where mp.code = ?1 or m.code = ?1 "
			+ "and (?2 is null or (m.storename like %?2% or mp.storename like %?2%))", countQuery = "select count(distinct m) from MerchantStore m left join m.parent mp "
					+ "where mp.code = ?1 or m.code = ?1 and (?2 is null or (m.storename like %?2% or mp.storename like %?2%))")
	Page<MerchantStore> listChilds(String storeCode, String storeName, Pageable pageable);

	@Query(value = "select * from MERCHANT_STORE m " + "where (m.STORE_CODE = ?1 or (?2 is null or m.PARENT_ID = ?2)) "
			+ "and (?3 is null or m.STORE_NAME like %?3%)", countQuery = "select count(*) from {h-schema}MERCHANT_STORE m where (m.STORE_CODE = ?1 or (?2 is null or m.PARENT_ID = ?2)) and (?3 is null or m.STORE_NAME like %?3%)", nativeQuery = true)
	Page<MerchantStore> listByGroup(String storeCode, Integer id, String storeName, Pageable pageable);

}



```
