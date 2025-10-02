# Criteria.java

## Review

## 1. Summary  
**Purpose**  
`com.salesmanager.core.model.common.Criteria` is a simple, mutable POJO that encapsulates query parameters used throughout the SalesManager backend. It supports both legacy (offset‑based) and new (page‑based) pagination, along with several filtering attributes (code, name, language, user, store identifiers, etc.).  

**Key Components**  
| Field | Role | Notes |
|-------|------|-------|
| `startIndex`, `maxCount` | Legacy offset‑based pagination | `legacyPagination` flag defaults to `true` |
| `startPage`, `pageSize` | New page‑based pagination | Default page size of 10 |
| `code`, `name`, `language`, `user`, `storeCode`, `search` | Simple filter criteria | Strings are nullable |
| `storeIds` | List of merchant store IDs for filtering | Uses `java.util.List<Integer>` |
| `orderBy`, `criteriaOrderByField` | Sorting | `CriteriaOrderBy` is an enum (`ASC`, `DESC`) |

The class follows the JavaBean convention, exposing getters and setters for all fields. It does **not** enforce immutability or validation – usage is driven by the surrounding DAO/Service layers.

**Design patterns & frameworks**  
- **Builder/Fluent API** – Not used; instead, plain setters are provided.  
- **JavaBean** – Enables easy mapping by ORM frameworks (e.g., Hibernate) or serialization libraries.  
- **Legacy vs. New Pagination** – The class keeps backward‑compatibility for older code that relies on `startIndex`/`maxCount`.

---

## 2. Detailed Description  
### Initialization  
All fields are initialized to sensible defaults (e.g., `startIndex = 0`, `pageSize = 10`, `orderBy = DESC`). No constructors are defined, so the default no‑arg constructor is implicit.

### Runtime Flow  
1. **Creation** – Client code instantiates `Criteria` (`new Criteria()`).  
2. **Configuration** – The caller populates fields via setters.  
   - Legacy code may set `startIndex`/`maxCount` and leave `legacyPagination` as `true`.  
   - New code can set `startPage`/`pageSize` and optionally switch `legacyPagination` to `false`.  
3. **Usage** – The populated `Criteria` is passed to a DAO or repository that translates it into a query (JPA/Hibernate, JDBC, or any custom query builder).  
4. **Cleanup** – No explicit cleanup is needed; the object is short‑lived or reused per request.

### Assumptions & Constraints  
- **Mutability** – The object can be modified at any time; callers must avoid sharing a single instance across threads without proper synchronization.  
- **Nullability** – Most string fields are nullable; the DAO layer must guard against `NullPointerException`.  
- **Pagination Interference** – The class does not enforce that either legacy or new pagination is used exclusively; if both are set, the DAO layer decides which to honor.  
- **Ordering** – `criteriaOrderByField` may be `null`; the DAO layer must default to a sensible column.

### Architecture & Design Choices  
- **Simplicity** – The class is intentionally lightweight, acting purely as a data holder.  
- **Legacy Support** – The dual pagination fields reflect a migration strategy.  
- **No Validation** – Validation is deferred to calling layers to keep the POJO generic.  

---

## 3. Functions/Methods  
| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `getMaxCount()` | Retrieve legacy max row count | None | `int` | None |
| `setMaxCount(int)` | Set legacy max row count | `maxCount` | None | Updates field |
| `getStartIndex()` | Retrieve legacy offset | None | `int` | None |
| `setStartIndex(int)` | Set legacy offset | `startIndex` | None | Updates field |
| `getCode()` | Get filter by code | None | `String` | None |
| `setCode(String)` | Set filter by code | `code` | None | Updates field |
| `setOrderBy(CriteriaOrderBy)` | Set sort direction | `orderBy` | None | Updates field |
| `getOrderBy()` | Get sort direction | None | `CriteriaOrderBy` | None |
| `getLanguage()` | Get language filter | None | `String` | None |
| `setLanguage(String)` | Set language filter | `language` | None | Updates field |
| `getUser()` | Get user filter | None | `String` | None |
| `setUser(String)` | Set user filter | `user` | None | Updates field |
| `getName()` | Get name filter | None | `String` | None |
| `setName(String)` | Set name filter | `name` | None | Updates field |
| `getCriteriaOrderByField()` | Get field to order by | None | `String` | None |
| `setCriteriaOrderByField(String)` | Set field to order by | `criteriaOrderByField` | None | Updates field |
| `getSearch()` | Get search term | None | `String` | None |
| `setSearch(String)` | Set search term | `search` | None | Updates field |
| `getStoreCode()` | Get store code filter | None | `String` | None |
| `setStoreCode(String)` | Set store code filter | `storeCode` | None | Updates field |
| `getPageSize()` | Get page size | None | `int` | None |
| `setPageSize(int)` | Set page size | `pageSize` | None | Updates field |
| `getStartPage()` | Get page number | None | `int` | None |
| `setStartPage(int)` | Set page number | `startPage` | None | Updates field |
| `isLegacyPagination()` | Check if legacy pagination is active | None | `boolean` | None |
| `setLegacyPagination(boolean)` | Toggle legacy pagination | `legacyPagination` | None | Updates field |
| `getStoreIds()` | Retrieve list of store IDs | None | `List<Integer>` | None |
| `setStoreIds(List<Integer>)` | Set list of store IDs | `storeIds` | None | Updates field |

All setters perform straightforward field assignment; there is no validation or transformation logic.

---

## 4. Dependencies  
| Dependency | Type | Role |
|------------|------|------|
| `java.util.List` | JDK | Stores list of store IDs |
| `com.salesmanager.core.model.merchant.MerchantStore` | JDK import (unused) | Appears unused – may be a leftover artifact |
| `com.salesmanager.core.model.common.CriteriaOrderBy` | Internal | Enum defining sort order (`ASC`, `DESC`) |

No third‑party libraries are referenced; the class is purely JDK‑based.

---

## 5. Additional Notes  

### Edge Cases & Potential Pitfalls  
- **Concurrent Modifications** – Because the class is mutable, sharing an instance across threads without synchronization can cause race conditions.  
- **Dual Pagination** – If both legacy and new pagination values are set, the DAO layer must decide which to honor. A silent fallback may lead to unexpected results.  
- **Null Fields** – DAO logic must treat `null` values gracefully; otherwise, NPEs or unintended query behavior may occur.  
- **Search Semantics** – The `search` field is a free‑text term; it is up to the DAO to decide how to apply it (e.g., `LIKE`, full‑text search).  
- **Unused Import** – `MerchantStore` is imported but never used; it can be removed to clean up the code.

### Possible Enhancements  
1. **Immutability** – Provide a builder or immutable version to avoid accidental mutation.  
2. **Validation** – Add basic checks (e.g., non‑negative page numbers, non‑zero page size).  
3. **Pagination Strategy Enum** – Encapsulate the choice between legacy and new pagination in an enum to make intent clearer.  
4. **Documentation** – Inline Javadoc on each field/method clarifying expected usage.  
5. **Deprecated Annotation** – Mark legacy pagination fields as `@Deprecated` once the system fully migrates.  
6. **Unit Tests** – Add tests for boundary conditions (e.g., negative page numbers).  

Overall, `Criteria` serves as a straightforward data holder for query parameters. Its simplicity is appropriate for its role, but adding immutability, validation, and clearer pagination handling would increase robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.common;

import java.util.List;

import com.salesmanager.core.model.merchant.MerchantStore;

public class Criteria {

	// legacy pagination
	private int startIndex = 0;
	private int maxCount = 0;
	// new pagination
	private int startPage = 0;
	private int pageSize = 10;
	private boolean legacyPagination = true;
	private String code;
	private String name;
	private String language;
	private String user;
	private String storeCode;
	private List<Integer> storeIds;

	private CriteriaOrderBy orderBy = CriteriaOrderBy.DESC;
	private String criteriaOrderByField;
	private String search;

	public int getMaxCount() {
		return maxCount;
	}

	public void setMaxCount(int maxCount) {
		this.maxCount = maxCount;
	}

	public int getStartIndex() {
		return startIndex;
	}

	public void setStartIndex(int startIndex) {
		this.startIndex = startIndex;
	}

	public String getCode() {
		return code;
	}

	public void setCode(String code) {
		this.code = code;
	}

	public void setOrderBy(CriteriaOrderBy orderBy) {
		this.orderBy = orderBy;
	}

	public CriteriaOrderBy getOrderBy() {
		return orderBy;
	}

	public String getLanguage() {
		return language;
	}

	public void setLanguage(String language) {
		this.language = language;
	}

	public String getUser() {
		return user;
	}

	public void setUser(String user) {
		this.user = user;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getCriteriaOrderByField() {
		return criteriaOrderByField;
	}

	public void setCriteriaOrderByField(String criteriaOrderByField) {
		this.criteriaOrderByField = criteriaOrderByField;
	}

	public String getSearch() {
		return search;
	}

	public void setSearch(String search) {
		this.search = search;
	}

	public String getStoreCode() {
		return storeCode;
	}

	public void setStoreCode(String storeCode) {
		this.storeCode = storeCode;
	}

	public int getPageSize() {
		return pageSize;
	}

	public void setPageSize(int pageSize) {
		this.pageSize = pageSize;
	}

	public int getStartPage() {
		return startPage;
	}

	public void setStartPage(int startPage) {
		this.startPage = startPage;
	}

	public boolean isLegacyPagination() {
		return legacyPagination;
	}

	public void setLegacyPagination(boolean legacyPagination) {
		this.legacyPagination = legacyPagination;
	}

	public List<Integer> getStoreIds() {
		return storeIds;
	}

	public void setStoreIds(List<Integer> storeIds) {
		this.storeIds = storeIds;
	}


}


```
