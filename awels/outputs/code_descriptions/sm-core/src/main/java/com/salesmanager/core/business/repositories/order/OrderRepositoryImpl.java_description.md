# OrderRepositoryImpl.java

## Review

## 1. Summary  

The `OrderRepositoryImpl` class implements custom query logic for retrieving `Order` entities from the database. It contains two public methods:

| Method | Purpose |
|--------|---------|
| `listByStore` | **Deprecated** – Retrieves orders for a given `MerchantStore` using a custom `OrderCriteria` (filters, paging, sorting). |
| `listOrders` | Current implementation – Retrieves orders using a richer set of filters, supports pagination via a helper, and returns an `OrderList` wrapper. |

Both methods build JPQL queries manually by concatenating strings, bind parameters, and execute two queries per call (one for the total count and one for the paged result set). The repository also uses a lot of eager `LEFT JOIN FETCH` clauses to pre‑fetch related entities (`orderTotal`, `orderProducts`, `orderAttributes`, etc.).

**Key patterns / libraries**

* JPA (`javax.persistence`) for persistence and query execution.
* Apache Commons `StringUtils` for null/empty checks.
* A custom helper (`RepositoryHelper`) to paginate the result set.
* Custom DTOs (`OrderList`, `GenericEntityList`) for returning results.

## 2. Detailed Description  

### Core Components & Interaction  

1. **EntityManager** – injected via `@PersistenceContext`. All queries are executed through this EM.  
2. **Criteria Object** – `OrderCriteria` contains all possible filter parameters (customer name, email, status, id, phone, pagination, ordering).  
3. **Query Construction** – Both methods assemble the JPQL query piece‑by‑piece using `StringBuilder`.  
4. **Parameter Binding** – After building the query strings, parameters are set on the `Query` objects.  
5. **Execution** –  
   * A count query (`SELECT COUNT(o) …`) is executed first to obtain the total number of matching records.  
   * If the count is non‑zero, a second query retrieves the actual `Order` entities, optionally limited by paging parameters.  
6. **Result Packaging** – The results are wrapped in an `OrderList`, setting `totalCount`, `orders`, and (in `listOrders`) `totalPages`.  

### Execution Flow  

* **`listByStore`**  
  1. Build `SELECT COUNT(o)` and `SELECT o` queries with appropriate `WHERE` clauses.  
  2. Execute count query → `count`.  
  3. If `count == 0`, return an empty `OrderList`.  
  4. Set pagination (`setFirstResult`, `setMaxResults`) – **bug**: uses `first + max` instead of `max`.  
  5. Execute object query → set `orders` and `totalCount`.  

* **`listOrders`**  
  1. Build count and object queries similarly but with a more complete filter set.  
  2. Execute count query → `count`.  
  3. If `count == 0`, return empty list.  
  4. Create a `GenericEntityList` (unused after pagination).  
  5. Paginate via `RepositoryHelper.paginateQuery(...)`.  
  6. Populate `OrderList` fields.  

### Assumptions & Constraints  

* The `Order` entity name is **exactly** `Order`.  
* JPQL is used; the code assumes the underlying provider (likely Hibernate).  
* `OrderStatus.valueOf` will not fail – the status string must match an enum constant.  
* All string parameters are surrounded by `%` for a *contains* match. No escaping of wildcards.  
* Pagination logic assumes the `OrderCriteria` always contains valid numeric indices.  

## 3. Functions/Methods  

| Method | Parameters | Return | Purpose | Side‑Effects |
|--------|------------|--------|---------|--------------|
| `listByStore(MerchantStore, OrderCriteria)` | `store`, `criteria` | `OrderList` | Deprecated method to fetch orders for a store with filters & paging. | None |
| `listOrders(MerchantStore, OrderCriteria)` | `store`, `criteria` | `OrderList` | Current method to fetch orders with filters, paging, and total pages. | None |
| `like(String)` | `q` | `String` | Helper to wrap a string in `%` wildcards. | None |

### Notes on Utility Methods  

* `like` is trivial but could be inlined; it does not escape `%` or `_`, which may lead to unintended matches if the input contains these characters.

## 4. Dependencies  

| External | Type | Notes |
|----------|------|-------|
| `javax.persistence.*` | Standard JPA | EntityManager, Query |
| `org.apache.commons.lang3.StringUtils` | Third‑party | String checks |
| `com.salesmanager.core.business.utils.RepositoryHelper` | Internal | Handles pagination |
| `com.salesmanager.core.model.*` | Internal | Domain model and DTOs |
| JPA provider (likely Hibernate) | Third‑party | Implicit via EM, `str()` usage |

No platform‑specific libraries are used; however, the code contains provider‑specific JPQL fragments (e.g., `str(o.id)`).

## 5. Additional Notes  

### Identified Issues  

| Area | Problem | Impact | Suggested Fix |
|------|---------|--------|---------------|
| **JPQL Syntax** | `like:nm`, `like:name`, etc. lack a space before the parameter. | Query parse errors at runtime. | Change to `like :nm` (note the space). |
| **Pagination Logic (listByStore)** | `setMaxResults(first + max)` should be `setMaxResults(max)`. | Over‑fetching, possible performance regression. | Use `objectQ.setMaxResults(max)` and keep `setFirstResult(first)`. |
| **Duplication of Orders** | Multiple `LEFT JOIN FETCH` clauses can generate duplicate `Order` rows when an order has many products. | Incorrect list size / duplicates; may need `DISTINCT`. | Add `SELECT DISTINCT o` or use `JOIN FETCH` only where needed. |
| **`str(o.id)`** | Hibernate‑specific function; may not work on other JPA providers. | Provider portability issue. | Use `CAST(o.id AS string)` or rework the filter to avoid string conversion. |
| **Null / Empty Criteria** | No validation for `criteria` being `null`. | NPE if caller passes `null`. | Add `Objects.requireNonNull(criteria)` or guard checks. |
| **Enum Conversion** | `OrderStatus.valueOf(criteria.getStatus().toUpperCase())` throws if status is invalid. | Runtime exception. | Validate with `EnumUtils.isValidEnum` or try‑catch and return empty list. |
| **`GenericEntityList` Usage** | Created but not fully utilized; only totalCount and totalPages are set. | Unnecessary object creation. | Either remove or integrate fully. |
| **Deprecated Annotation** | `listByStore` is marked as deprecated in Javadoc but not with `@Deprecated`. | Documentation mismatch. | Add `@Deprecated` annotation. |
| **Logging / Metrics** | No logging or metrics. | Hard to debug query performance or failures. | Add SLF4J logs around query execution. |
| **String Escaping** | No escape of `%` or `_` in user input. | False positives/negatives in LIKE filters. | Escape special characters or use parameter binding with `escape` clause. |

### Potential Enhancements  

1. **Switch to JPA Criteria API**  
   * Safer, type‑checked query construction.  
   * Simplifies handling of optional filters.  

2. **Named Queries**  
   * Define reusable JPQL queries in the `Order` entity or `orm.xml`.  
   * Improves readability and performance (pre‑compiled).  

3. **DTO Projection**  
   * Instead of eager fetching all associations, project only the needed columns and use `JOIN FETCH` selectively.  

4. **Paging via `Pageable`**  
   * Adopt Spring Data’s `Pageable` abstraction for consistency.  

5. **Unit Tests**  
   * Add integration tests to verify JPQL syntax, parameter binding, and paging logic.  

6. **Error Handling**  
   * Wrap query exceptions in a custom `RepositoryException`.  

7. **Code Cleanup**  
   * Rename builders (`countQuery`, `entityQuery`) for clarity.  
   * Remove deprecated method entirely or keep it in a separate legacy class.  

Overall, the repository fulfills its functional requirements but suffers from several syntactic bugs, potential portability issues, and design choices that could be modernised and hardened with standard JPA practices. Addressing the above points will result in a more robust, maintainable, and portable data access layer.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.repositories.order;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;

import org.apache.commons.lang3.StringUtils;

import com.salesmanager.core.business.utils.RepositoryHelper;
import com.salesmanager.core.model.common.CriteriaOrderBy;
import com.salesmanager.core.model.common.GenericEntityList;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.OrderCriteria;
import com.salesmanager.core.model.order.OrderList;
import com.salesmanager.core.model.order.orderstatus.OrderStatus;


public class OrderRepositoryImpl implements OrderRepositoryCustom {

	
    @PersistenceContext
    private EntityManager em;
    
    /**
     * @deprecated
     */
	@SuppressWarnings("unchecked")
	@Override
	public OrderList listByStore(MerchantStore store, OrderCriteria criteria) {
		

		OrderList orderList = new OrderList();
		StringBuilder countBuilderSelect = new StringBuilder();
		StringBuilder objectBuilderSelect = new StringBuilder();
		
		String orderByCriteria = " order by o.id desc";
		
		if(criteria.getOrderBy()!=null) {
			if(CriteriaOrderBy.ASC.name().equals(criteria.getOrderBy().name())) {
				orderByCriteria = " order by o.id asc";
			}
		}
		
		String countBaseQuery = "select count(o) from Order as o";
		String baseQuery = "select o from Order as o left join fetch o.orderTotal ot left join fetch o.orderProducts op left join fetch o.orderAttributes oa left join fetch op.orderAttributes opo left join fetch op.prices opp";
		countBuilderSelect.append(countBaseQuery);
		objectBuilderSelect.append(baseQuery);

		
		
		StringBuilder countBuilderWhere = new StringBuilder();
		StringBuilder objectBuilderWhere = new StringBuilder();
		String whereQuery = " where o.merchant.id=:mId";
		countBuilderWhere.append(whereQuery);
		objectBuilderWhere.append(whereQuery);
		

		if(!StringUtils.isBlank(criteria.getCustomerName())) {
			String nameQuery =" and o.billing.firstName like:nm or o.billing.lastName like:nm";
			countBuilderWhere.append(nameQuery);
			objectBuilderWhere.append(nameQuery);
		}
		
		if(!StringUtils.isBlank(criteria.getPaymentMethod())) {
			String paymentQuery =" and o.paymentModuleCode like:pm";
			countBuilderWhere.append(paymentQuery);
			objectBuilderWhere.append(paymentQuery);
		}
		
		if(criteria.getCustomerId()!=null) {
			String customerQuery =" and o.customerId =:cid";
			countBuilderWhere.append(customerQuery);
			objectBuilderWhere.append(customerQuery);
		}
		
		objectBuilderWhere.append(orderByCriteria);
		

		//count query
		Query countQ = em.createQuery(
				countBuilderSelect.toString() + countBuilderWhere.toString());
		
		//object query
		Query objectQ = em.createQuery(
				objectBuilderSelect.toString() + objectBuilderWhere.toString());

		countQ.setParameter("mId", store.getId());
		objectQ.setParameter("mId", store.getId());
		

		if(!StringUtils.isBlank(criteria.getCustomerName())) {
			String nameParam = new StringBuilder().append("%").append(criteria.getCustomerName()).append("%").toString();
			countQ.setParameter("nm",nameParam);
			objectQ.setParameter("nm",nameParam);
		}
		
		if(!StringUtils.isBlank(criteria.getPaymentMethod())) {
			String payementParam = new StringBuilder().append("%").append(criteria.getPaymentMethod()).append("%").toString();
			countQ.setParameter("pm",payementParam);
			objectQ.setParameter("pm",payementParam);
		}
		
		if(criteria.getCustomerId()!=null) {
			countQ.setParameter("cid", criteria.getCustomerId());
			objectQ.setParameter("cid",criteria.getCustomerId());
		}
		

		Number count = (Number) countQ.getSingleResult();

		orderList.setTotalCount(count.intValue());
		
        if(count.intValue()==0)
        	return orderList;
        
		//TO BE USED
        int max = criteria.getMaxCount();
        int first = criteria.getStartIndex();
        
        objectQ.setFirstResult(first);
        
        
        
    	if(max>0) {
    			int maxCount = first + max;

			objectQ.setMaxResults(Math.min(maxCount, count.intValue()));
    	}
		
    	orderList.setOrders(objectQ.getResultList());

		return orderList;
		
		
	}

	@Override
	public OrderList listOrders(MerchantStore store, OrderCriteria criteria) {
		OrderList orderList = new OrderList();
		StringBuilder countBuilderSelect = new StringBuilder();
		StringBuilder objectBuilderSelect = new StringBuilder();

		String orderByCriteria = " order by o.id desc";

		if(criteria.getOrderBy()!=null) {
			if(CriteriaOrderBy.ASC.name().equals(criteria.getOrderBy().name())) {
				orderByCriteria = " order by o.id asc";
			}
		}

		
		String baseQuery = "select o from Order as o left join fetch o.delivery.country left join fetch o.delivery.zone left join fetch o.billing.country left join fetch o.billing.zone left join fetch o.orderTotal ot left join fetch o.orderProducts op left join fetch o.orderAttributes oa left join fetch op.orderAttributes opo left join fetch op.prices opp";
		String countBaseQuery = "select count(o) from Order as o";
		
		countBuilderSelect.append(countBaseQuery);
		objectBuilderSelect.append(baseQuery);

		StringBuilder objectBuilderWhere = new StringBuilder();

		String storeQuery =" where o.merchant.code=:mCode";
		objectBuilderWhere.append(storeQuery);
		countBuilderSelect.append(storeQuery);
		
		if(!StringUtils.isEmpty(criteria.getCustomerName())) {
			String nameQuery =  " and o.billing.firstName like:name or o.billing.lastName like:name";
			objectBuilderWhere.append(nameQuery);
			countBuilderSelect.append(nameQuery);
		}
		
		if(!StringUtils.isEmpty(criteria.getEmail())) {
			String nameQuery =  " and o.customerEmailAddress like:email";
			objectBuilderWhere.append(nameQuery);
			countBuilderSelect.append(nameQuery);
		}
		
		//id
		if(criteria.getId() != null) {
			String nameQuery =  " and str(o.id) like:id";
			objectBuilderWhere.append(nameQuery);
			countBuilderSelect.append(nameQuery);
		}
		
		//phone
		if(!StringUtils.isEmpty(criteria.getCustomerPhone())) {
			String nameQuery =  " and o.billing.telephone like:phone or o.delivery.telephone like:phone";
			objectBuilderWhere.append(nameQuery);
			countBuilderSelect.append(nameQuery);
		}
		
		//status
		if(!StringUtils.isEmpty(criteria.getStatus())) {
			String nameQuery =  " and o.status =:status";
			objectBuilderWhere.append(nameQuery);
			countBuilderSelect.append(nameQuery);
		}
	
		objectBuilderWhere.append(orderByCriteria);

		//count query
		Query countQ = em.createQuery(
				countBuilderSelect.toString());

		//object query
		Query objectQ = em.createQuery(
				objectBuilderSelect.toString() + objectBuilderWhere.toString());
		
		//customer name
		if(!StringUtils.isEmpty(criteria.getCustomerName())) {
			countQ.setParameter("name", like(criteria.getCustomerName()));
			objectQ.setParameter("name", like(criteria.getCustomerName()));
		}
		
		//email
		if(!StringUtils.isEmpty(criteria.getEmail())) {
			countQ.setParameter("email", like(criteria.getEmail()));
			objectQ.setParameter("email", like(criteria.getEmail()));			
		}
		
		//id
		if(criteria.getId() != null) {
			countQ.setParameter("id", like(String.valueOf(criteria.getId())));
			objectQ.setParameter("id", like(String.valueOf(criteria.getId())));
		}
		
		//phone
		if(!StringUtils.isEmpty(criteria.getCustomerPhone())) {
			countQ.setParameter("phone", like(criteria.getCustomerPhone()));
			objectQ.setParameter("phone", like(criteria.getCustomerPhone()));
		}
		
		//status
		if(!StringUtils.isEmpty(criteria.getStatus())) {
			countQ.setParameter("status", OrderStatus.valueOf(criteria.getStatus().toUpperCase()));
			objectQ.setParameter("status", OrderStatus.valueOf(criteria.getStatus().toUpperCase()));
		}
		

		countQ.setParameter("mCode", store.getCode());
		objectQ.setParameter("mCode", store.getCode());


		Number count = (Number) countQ.getSingleResult();

		if(count.intValue()==0)
			return orderList;

	    @SuppressWarnings("rawtypes")
		GenericEntityList entityList = new GenericEntityList();
	    entityList.setTotalCount(count.intValue());
		
		objectQ = RepositoryHelper.paginateQuery(objectQ, count, entityList, criteria);
		
		//TODO use GenericEntityList

		orderList.setTotalCount(entityList.getTotalCount());
		orderList.setTotalPages(entityList.getTotalPages());

		orderList.setOrders(objectQ.getResultList());

		return orderList;
	}
	
	private String like(String q) {
		return '%' + q + '%';
	}


}



```
