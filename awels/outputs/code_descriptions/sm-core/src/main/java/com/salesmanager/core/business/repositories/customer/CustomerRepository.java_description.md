# CustomerRepository.java

## Review

## 1. Summary

| Item | Description |
|------|-------------|
| **Purpose** | The `CustomerRepository` interface provides CRUD‑like access to `Customer` entities with a large amount of eager fetching. It is meant to be used by the **SalesManager** e‑commerce core module. |
| **Key components** | - Extends `JpaRepository<Customer, Long>` for basic persistence.<br>- Extends `CustomerRepositoryCustom` for bespoke queries that cannot be expressed declaratively.<br>- A collection of `@Query`‑annotated JPQL methods that eagerly join related entities (`merchantStore`, `defaultLanguage`, `attributes`, `customerOption`, `groups`, `billing`, `delivery`, `country`, `zone`, etc.). |
| **Design patterns / frameworks** | - **Repository pattern** via Spring Data JPA.<br>- Declarative query building with **JPQL** and **@Query** annotations.<br>- **Custom repository interface** pattern for complex logic. |
| **Observations** | The repository is highly “fetch‑heavy” – almost every read method pulls a full graph of related entities. This can have serious performance implications, especially if not all callers need the entire graph. |


## 2. Detailed Description

### Execution flow

1. **Initialization** – Spring Data creates a dynamic implementation at startup.  
2. **Runtime** – When a method such as `findByNick` is invoked, Spring executes the JPQL query, mapping results back to the `Customer` entity graph.  
3. **Cleanup** – The repository is stateless; the EntityManager handles transaction boundaries per Spring configuration.

### Interaction of components

| Component | Interaction |
|-----------|-------------|
| `JpaRepository` | Provides standard CRUD methods (save, delete, findById, etc.). |
| `CustomerRepositoryCustom` | Holds additional custom methods implemented in a separate class (`CustomerRepositoryImpl`). |
| `@Query` methods | Override default fetch behaviour, ensuring eager loading of related associations. |

### Assumptions & Constraints

- All read operations are expected to require a *complete* `Customer` graph.  
- The underlying database is relational (likely MySQL/PostgreSQL) and supports JPQL.  
- The entity mappings (`Customer`, `MerchantStore`, etc.) are properly annotated for JPA/Hibernate.  
- The `storeId` and `store` parameters correspond to unique identifiers of a `MerchantStore`.

### Design choices

- **Manual fetch joins**: The developer chose to write explicit join‑fetch JPQL instead of relying on default `FetchType.EAGER` or `@EntityGraph`.  
- **Method overloading**: Several `findByNick` methods share the same name but different parameter lists, allowing callers to specify store context or a reset‑token.  
- **Explicit `distinct`**: Used to collapse duplicates that arise from multiple left joins.

These choices give full control over SQL generation but increase boilerplate and risk of errors (e.g., missing a join, typo in an alias).  


## 3. Functions/Methods

| Method | Purpose | Inputs | Output | Side Effects |
|--------|---------|--------|--------|--------------|
| `Customer findOne(Long id)` | Retrieve a single `Customer` with all related entities eagerly loaded. | `Long id` | `Customer` (or `null` if not found) | None |
| `List<Customer> findByName(String name)` | Find customers whose *billing* first name matches `name`. | `String name` | `List<Customer>` | None |
| `Customer findByNick(String nick)` | Retrieve a customer by its unique nick. | `String nick` | `Customer` | None |
| `Customer findByNick(String nick, int storeId)` | Same as above but scoped to a particular store ID. | `String nick`, `int storeId` | `Customer` | None |
| `Customer findByNick(String nick, String store)` | Same as above but scoped to a store code. | `String nick`, `String store` | `Customer` | None |
| `Customer findByResetPasswordToken(String token, String store)` | Fetch a customer who requested a password reset identified by `token` within a specific store. | `String token`, `String store` | `Customer` | None |
| `List<Customer> findByStore(int storeId)` | Retrieve all customers belonging to a given store. | `int storeId` | `List<Customer>` | None |

### Utility / Reusable aspects

- All queries share a *base* fetch join pattern that could be extracted into an `@EntityGraph` to avoid duplication.  
- No explicit helper methods; the heavy lifting is done in the JPQL strings themselves.

## 4. Dependencies

| Library / Framework | Role | Standard / 3rd‑party |
|---------------------|------|----------------------|
| Spring Data JPA (`org.springframework.data.jpa.repository.JpaRepository`, `@Query`) | Repository abstraction & JPQL execution | 3rd‑party |
| JPA / Hibernate | ORM mapping and entity persistence | 3rd‑party |
| Java Persistence API (JPA) annotations | Entity relationships, join fetch | Standard |
| Underlying database (MySQL/PostgreSQL) | Storage | Platform-specific (but database‑agnostic via JPA) |

No direct HTTP, messaging, or caching dependencies are referenced here.

## 5. Additional Notes & Recommendations

### Pros

- **Explicit control** over SQL generation (join fetch, aliasing).  
- **Overloaded methods** provide flexible query entry points.  
- **Custom repository interface** allows complex logic outside of JPQL.

### Cons / Risks

1. **Over‑fetching / N+1**  
   - Every query pulls a large object graph. If callers only need basic customer data, this leads to unnecessary joins, larger result sets, and higher memory usage.  
2. **Maintenance burden**  
   - The same join logic is repeated in ~10 methods. A typo in one alias can break multiple queries.  
3. **Potential query duplication**  
   - Some queries are identical except for a `where` clause (e.g., by nick vs. by reset token). A shared method with a parameter for the `where` clause could reduce duplication.  
4. **`findOne` name collision**  
   - `JpaRepository` already exposes `findById` returning `Optional<T>`. Renaming or explicitly using `Optional<Customer>` would align better with modern Spring conventions.  
5. **Method naming inconsistency**  
   - `findByName` actually queries `billing.firstName`. A name like `findByBillingFirstName` would be clearer.  
6. **Nullability handling**  
   - None of the methods return `Optional`; callers must check for `null`.  
7. **Transaction boundaries**  
   - All queries are read‑only, but the repository does not annotate them with `@Transactional(readOnly = true)`. While Spring Data applies a default, being explicit can prevent accidental write operations.  

### Suggested Improvements

| Change | Benefit |
|--------|---------|
| **Use `@EntityGraph`** instead of manual join fetches. Example: `@EntityGraph(attributePaths = {"merchantStore","defaultLanguage","attributes","attributes.customerOption","attributes.customerOptionValue","groups","delivery","billing"})` | Reduces duplication, easier to maintain, automatically applied by Spring. |
| **Rename `findOne` → `findById` with `Optional<Customer>`** | Avoids conflict with `JpaRepository`'s default method; embraces Optional for null safety. |
| **Add `@Transactional(readOnly = true)`** to all query methods | Signals intent, may improve performance via Hibernate. |
| **Abstract common query string** into a constant or utility method to avoid duplication. | Easier refactoring and less risk of mismatches. |
| **Consider lazy loading for rarely used associations** (e.g., `customerOption`, `customerOptionValue`) | Cuts down join load when not needed. |
| **Document intent in method names** (e.g., `findByBillingFirstName`). | Improves readability for future developers. |
| **Add `@Param` annotations** to query parameters for clarity. | Avoids reliance on positional parameters (`?1`, `?2`), which are fragile if the query changes. |

### Edge Cases

- **Null `nick` or `token`**: JPQL `where` clause will fail if `nick` or `token` is `null`. Consider adding `?1 IS NULL` or using `CriteriaBuilder` for safer handling.  
- **Multiple customers sharing same nick**: Queries do not enforce uniqueness; if the DB allows duplicates, a single result might not be deterministic.  
- **Large result sets** (e.g., `findByStore` on a store with millions of customers): May exhaust memory; pagination (`Pageable`) would be preferable.  

### Future Enhancements

- **Paging & Sorting**: Add `Page<Customer>` return types for methods that could return many records (`findByStore`, `findByName`).  
- **Batching**: If customers are frequently accessed in bulk, consider `EntityManager` batch fetch or `@BatchSize` hints.  
- **Cache**: Read‑only data such as `Customer` lookup by nick could be cached (e.g., with Spring Cache) to reduce DB load.  
- **DTO projection**: For endpoints that only need a subset of fields, use Spring Data's interface/tuple projections to avoid full entity loading.  

---

**Bottom line** – The repository is functional and gives fine‑grained control over eager fetching, but it is verbose and can be fragile. Refactoring to use `@EntityGraph`, improving method naming, and embracing Spring Data conventions (e.g., `Optional`, pagination) would make the code cleaner, more maintainable, and potentially more performant.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.repositories.customer;

import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import com.salesmanager.core.model.customer.Customer;

public interface CustomerRepository extends JpaRepository<Customer, Long>, CustomerRepositoryCustom {

	
	@Query("select c from Customer c join fetch c.merchantStore cm left join fetch c.defaultLanguage cl left join fetch c.attributes ca left join fetch ca.customerOption cao left join fetch ca.customerOptionValue cav left join fetch cao.descriptions caod left join fetch cav.descriptions left join fetch c.groups where c.id = ?1")
	Customer findOne(Long id);
	
	@Query("select distinct c from Customer c join fetch c.merchantStore cm left join fetch c.defaultLanguage cl left join fetch c.attributes ca left join fetch ca.customerOption cao left join fetch ca.customerOptionValue cav left join fetch cao.descriptions caod left join fetch cav.descriptions left join fetch c.groups  where c.billing.firstName = ?1")
	List<Customer> findByName(String name);
	
	@Query("select c from Customer c join fetch c.merchantStore cm left join fetch c.defaultLanguage cl left join fetch c.attributes ca left join fetch ca.customerOption cao left join fetch ca.customerOptionValue cav left join fetch cao.descriptions caod left join fetch cav.descriptions left join fetch c.groups  where c.nick = ?1")
	Customer findByNick(String nick);
	
	@Query("select c from Customer c "
			+ "join fetch c.merchantStore cm "
			+ "left join fetch c.defaultLanguage cl "
			+ "left join fetch c.attributes ca "
			+ "left join fetch ca.customerOption cao "
			+ "left join fetch ca.customerOptionValue cav "
			+ "left join fetch cao.descriptions caod "
			+ "left join fetch cav.descriptions  "
			+ "left join fetch c.groups  "
			+ "left join fetch c.delivery cd "
			+ "left join fetch c.billing cb "
			+ "left join fetch cd.country "
			+ "left join fetch cd.zone "
			+ "left join fetch cb.country "
			+ "left join fetch cb.zone "
			+ "where c.nick = ?1 and cm.id = ?2")
	Customer findByNick(String nick, int storeId);
	
	@Query("select c from Customer c "
			+ "join fetch c.merchantStore cm "
			+ "left join fetch c.defaultLanguage cl "
			+ "left join fetch c.attributes ca "
			+ "left join fetch ca.customerOption cao "
			+ "left join fetch ca.customerOptionValue cav "
			+ "left join fetch cao.descriptions caod "
			+ "left join fetch cav.descriptions  "
			+ "left join fetch c.groups  "
			+ "left join fetch c.delivery cd "
			+ "left join fetch c.billing cb "
			+ "left join fetch cd.country "
			+ "left join fetch cd.zone "
			+ "left join fetch cb.country "
			+ "left join fetch cb.zone "
			+ "where c.nick = ?1 and cm.code = ?2")
	Customer findByNick(String nick, String store);
	
	@Query("select c from Customer c "
			+ "join fetch c.merchantStore cm "
			+ "left join fetch c.defaultLanguage cl "
			+ "left join fetch c.attributes ca "
			+ "left join fetch ca.customerOption cao "
			+ "left join fetch ca.customerOptionValue cav "
			+ "left join fetch cao.descriptions caod "
			+ "left join fetch cav.descriptions  "
			+ "left join fetch c.groups  "
			+ "left join fetch c.delivery cd "
			+ "left join fetch c.billing cb "
			+ "left join fetch cd.country "
			+ "left join fetch cd.zone "
			+ "left join fetch cb.country "
			+ "left join fetch cb.zone "
			+ "where c.credentialsResetRequest.credentialsRequest = ?1 and cm.code = ?2")
	Customer findByResetPasswordToken(String token, String store);
	
	@Query("select distinct c from Customer c join fetch c.merchantStore cm left join fetch c.defaultLanguage cl left join fetch c.attributes ca left join fetch ca.customerOption cao left join fetch ca.customerOptionValue cav left join fetch cao.descriptions caod left join fetch cav.descriptions left join fetch c.groups  where cm.id = ?1")
	List<Customer> findByStore(int storeId);
	

}



```
