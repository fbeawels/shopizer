# CustomerRepositoryImpl.java

## Review

## 1. Summary

The file implements a **custom JPA repository** for the `Customer` entity, providing a method to list customers by store while applying a rich set of filtering criteria (`CustomerCriteria`).  
Key points:

- **Purpose**: Return a paginated list of customers for a specific merchant store, applying filters such as name, first/last name, email, and country.  
- **Core Components**:
  * `CustomerRepositoryImpl` – concrete class that implements `CustomerRepositoryCustom`.
  * `listByStore(MerchantStore, CustomerCriteria)` – builds two JPQL queries (count & entity retrieval), sets parameters, and handles pagination.
- **Design Patterns / Libraries**:
  * **Custom Repository Pattern** (Spring Data style) – provides additional methods beyond CRUD.
  * **JPQL** – plain string concatenation rather than Criteria API.
  * **Apache Commons Lang** – `StringUtils.isBlank` for null/empty checks.
  * Uses JPA’s `EntityManager` via `@PersistenceContext`.

## 2. Detailed Description

### Flow of Execution

1. **Initialization**  
   The `EntityManager` (`em`) is injected by Spring/JPA.  
   `CustomerList` is instantiated to hold the final result.

2. **Query Construction**  
   *Base Queries* – Two `StringBuilder`s hold the base SELECT clauses:  
   - `baseCountQuery` for `SELECT COUNT(c) FROM Customer c`.  
   - `baseQuery` for retrieving full `Customer` objects, with several `LEFT JOIN FETCH` clauses to eagerly load related entities (`delivery`, `billing`, `attributes`, etc.).  
   *WHERE Clauses* – A `whereQuery` ensures the customer belongs to the supplied `MerchantStore`.  
   Conditional blocks append extra predicates when corresponding `CustomerCriteria` fields are non‑blank.  
   The same WHERE string is appended to both the count and entity queries.

3. **Parameter Binding**  
   Parameters (`mId`, `nm`, `fn`, `ln`, `email`, `country`) are set only if the associated criterion is present.  
   LIKE patterns are wrapped with `%` for partial matching.

4. **Execution**  
   *Count Query* → returns a single integer (`count`).  
   *Entity Query* → paginated via `setFirstResult` and `setMaxResults`.

5. **Result Packaging**  
   The `CustomerList` is populated with `totalCount` and the list of retrieved customers (or empty if `count == 0`).

### Assumptions & Constraints

- `criteria` and `store` are assumed non‑null.  
- The `Customer` entity has the mapped relationships expected by the JPQL (`billing`, `delivery`, `attributes`, etc.).  
- Pagination is controlled by `criteria.getStartIndex()` and `criteria.getMaxCount()`.  
- The application uses a JPA provider that supports the JPQL syntax used (e.g., Hibernate).

### Architecture & Design Choices

- **String‑based JPQL**: Offers quick query writing but is prone to typos and lacks type safety.  
- **Custom Repository**: Allows adding complex business logic outside the default CRUD methods.  
- **Eager Fetching**: `LEFT JOIN FETCH` is used to avoid N+1 select problems when accessing related entities later.  
- **Manual Pagination**: The logic calculates the max results manually, which could be simplified by using JPA’s `Pageable` abstraction.

## 3. Functions/Methods

| Method | Purpose | Parameters | Return Type | Side Effects |
|--------|---------|------------|-------------|--------------|
| `listByStore(MerchantStore store, CustomerCriteria criteria)` | Builds and executes JPQL queries to fetch customers filtered by store and criteria, then returns a `CustomerList` containing the total count and the paginated list of customers. | `store` – the merchant store to filter by.<br>`criteria` – filter values (name, first/last name, email, country). | `CustomerList` – wrapper containing `totalCount` and `List<Customer>` (may be empty). | None directly; may modify the internal state of the returned `CustomerList`. |

### Reusable / Utility Methods
None defined in this class; all logic is inlined within `listByStore`.  Refactoring could move common predicate building or parameter setting into private helper methods for clarity and reuse.

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.EntityManager` | JPA | Core persistence API. |
| `javax.persistence.PersistenceContext` | JPA | Injects the `EntityManager`. |
| `javax.persistence.Query` | JPA | For executing JPQL. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | Utility for checking blank strings. |
| `com.salesmanager.core.model.customer.CustomerCriteria` | Internal | Holds filter values. |
| `com.salesmanager.core.model.customer.CustomerList` | Internal | DTO for paginated results. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Internal | Represents the store context. |

No external frameworks beyond JPA/Hibernate and Apache Commons are used. The code is platform‑agnostic, assuming a Java EE / Spring environment with JPA configured.

## 5. Additional Notes

### Edge Cases & Issues

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Typo in JPQL** – `"and c..billing.firstName"` (double dot). | Compile‑time or runtime JPQL syntax error. | Remove the extra dot: `c.billing.firstName`. |
| **Missing space before parameter** – `"like:nm"` should be `"like :nm"`. | JPQL parser will fail. | Add a space: `like :nm`. |
| **Missing space before parameters in other predicates** (`like:fn`, `like:email`, `like:country`). | Same as above. | Add spaces. |
| **Potential N+1** – If the relationships are lazy, repeated fetches may still occur. | Performance degradation. | Keep `LEFT JOIN FETCH` or use batch fetching. |
| **Duplicate `countQ` and `objectQ` building** – Lots of duplicated code. | Hard to maintain. | Extract common parts into helper methods. |
| **Pagination logic** – `setMaxResults(Math.min(maxCount, count.intValue()))` actually sets the **max number of rows to return** to the smaller of `(first+max)` and `count`, but `setMaxResults` expects a limit, not an end index. | May return more or fewer rows than intended. | Use `objectQ.setMaxResults(max)` if `max > 0`. |
| **Null `criteria` or `store`** – No null‑check. | NullPointerException. | Validate arguments at method start. |
| **`customerList.setCustomers(objectQ.getResultList());`** – If `count==0`, this method returns early; otherwise it sets a list. If `objectQ.getResultList()` returns `null`, this can lead to `NullPointerException`. | Unexpected behavior. | Ensure the list is never null, e.g., initialize with an empty list. |
| **`CustomerList` may expose mutable internal state** – Not an issue here but consider defensive copying. | Encapsulation risk. | Return immutable collections or copy. |
| **`count.intValue()` cast** – If the count is large, `int` may overflow. | Wrong pagination bounds. | Use `long` for counts and `int` for pagination indexes. |

### Potential Enhancements

1. **Switch to Criteria API or Spring Data JPA’s `Specification`** – provides type safety and easier dynamic query building.
2. **Use `Pageable` / `Page`** – Spring Data offers a standard pagination abstraction that would simplify the code.
3. **Parameter Validation** – Add explicit checks (`Objects.requireNonNull`) to fail fast with clear messages.
4. **Refactor Predicate Building** – Extract predicate construction into a separate method or builder to reduce duplication and improve readability.
5. **Error Handling** – Wrap query execution in try/catch and translate JPA exceptions to application‑specific ones.
6. **Logging** – Log constructed JPQL for debugging, but mask sensitive data.
7. **Unit Tests** – Add tests covering all filter combinations, pagination, and edge cases.

### Final Thoughts

The implementation achieves its core goal—returning a filtered, paginated list of customers—but suffers from several syntactic bugs and maintainability concerns. Addressing the typos, simplifying the pagination logic, and refactoring repetitive code will make the repository robust and easier to extend. Transitioning to a more modern query‑building approach (Criteria API or Spring Data `Specification`) would also reduce the risk of future errors.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.repositories.customer;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;

import org.apache.commons.lang3.StringUtils;

import com.salesmanager.core.model.customer.CustomerCriteria;
import com.salesmanager.core.model.customer.CustomerList;
import com.salesmanager.core.model.merchant.MerchantStore;


public class CustomerRepositoryImpl implements CustomerRepositoryCustom {

	
    @PersistenceContext
    private EntityManager em;
    
	@SuppressWarnings("unchecked")
	@Override
	public CustomerList listByStore(MerchantStore store, CustomerCriteria criteria) {
		

		CustomerList customerList = new CustomerList();
		StringBuilder countBuilderSelect = new StringBuilder();
		StringBuilder objectBuilderSelect = new StringBuilder();
		
		String baseCountQuery = "select count(c) from Customer as c";
		String baseQuery = "select c from Customer as c  left join fetch c.delivery.country left join fetch c.delivery.zone left join fetch c.billing.country left join fetch c.billing.zone";
		countBuilderSelect.append(baseCountQuery);
		objectBuilderSelect.append(baseQuery);
		
		StringBuilder countBuilderWhere = new StringBuilder();
		StringBuilder objectBuilderWhere = new StringBuilder();
		String whereQuery = " where c.merchantStore.id=:mId";
		countBuilderWhere.append(whereQuery);
		objectBuilderWhere.append(whereQuery);

		if(!StringUtils.isBlank(criteria.getName())) {
			String nameQuery =" and c.billing.firstName like:nm or c.billing.lastName like:nm";
			countBuilderWhere.append(nameQuery);
			objectBuilderWhere.append(nameQuery);
		}
		
		if(!StringUtils.isBlank(criteria.getFirstName())) {
			String nameQuery =" and c..billing.firstName like:fn";
			countBuilderWhere.append(nameQuery);
			objectBuilderWhere.append(nameQuery);
		}
		
		if(!StringUtils.isBlank(criteria.getLastName())) {
			String nameQuery =" and c.billing.lastName like:ln";
			countBuilderWhere.append(nameQuery);
			objectBuilderWhere.append(nameQuery);
		}
		
		if(!StringUtils.isBlank(criteria.getEmail())) {
			String mailQuery =" and c.emailAddress like:email";
			countBuilderWhere.append(mailQuery);
			objectBuilderWhere.append(mailQuery);
		}
		
		if(!StringUtils.isBlank(criteria.getCountry())) {
			String countryQuery =" and c.billing.country.isoCode like:country";
			countBuilderWhere.append(countryQuery);
			objectBuilderWhere.append(countryQuery);
		}
		
		objectBuilderSelect.append(" left join fetch c.attributes ca left join fetch ca.customerOption cao left join fetch ca.customerOptionValue cav left join fetch cao.descriptions caod left join fetch cav.descriptions  left join fetch c.groups");

		//count query
		Query countQ = em.createQuery(
				countBuilderSelect.toString() + countBuilderWhere.toString());
		
		//object query
		Query objectQ = em.createQuery(
				objectBuilderSelect.toString() + objectBuilderWhere.toString());

		countQ.setParameter("mId", store.getId());
		objectQ.setParameter("mId", store.getId());
		

		if(!StringUtils.isBlank(criteria.getName())) {
			String nameParam = new StringBuilder().append("%").append(criteria.getName()).append("%").toString();
			countQ.setParameter("nm",nameParam);
			objectQ.setParameter("nm",nameParam);
		}
		
		if(!StringUtils.isBlank(criteria.getFirstName())) {
			String nameParam = new StringBuilder().append("%").append(criteria.getFirstName()).append("%").toString();
			countQ.setParameter("fn",nameParam);
			objectQ.setParameter("fn",nameParam);
		}
		
		if(!StringUtils.isBlank(criteria.getLastName())) {
			String nameParam = new StringBuilder().append("%").append(criteria.getLastName()).append("%").toString();
			countQ.setParameter("ln",nameParam);
			objectQ.setParameter("ln",nameParam);
		}
		
		if(!StringUtils.isBlank(criteria.getEmail())) {
			String email = new StringBuilder().append("%").append(criteria.getEmail()).append("%").toString();
			countQ.setParameter("email",email);
			objectQ.setParameter("email",email);
		}
		
		if(!StringUtils.isBlank(criteria.getCountry())) {
			String country = new StringBuilder().append("%").append(criteria.getCountry()).append("%").toString();
			countQ.setParameter("country",country);
			objectQ.setParameter("country",country);
		}
		
		


		Number count = (Number) countQ.getSingleResult();

		customerList.setTotalCount(count.intValue());
		
        if(count.intValue()==0)
        	return customerList;
        
		//TO BE USED
        int max = criteria.getMaxCount();
        int first = criteria.getStartIndex();
        
        objectQ.setFirstResult(first);
        
        
        
    	if(max>0) {
    			int maxCount = first + max;

			objectQ.setMaxResults(Math.min(maxCount, count.intValue()));
    	}
		
		customerList.setCustomers(objectQ.getResultList());

		return customerList;
		
		
	}

    

}



```
