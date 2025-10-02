# CustomerReviewServiceImpl.java

## Review

## 1. Summary  
The **`CustomerReviewServiceImpl`** is a Spring‐managed service that handles CRUD‑like operations for `CustomerReview` entities while also maintaining aggregate rating data on the reviewed customer.  
* **Key responsibilities**  
  * Persist new or updated reviews.  
  * Recalculate the reviewed customer’s average rating and review count whenever a review is created or updated.  
  * Provide convenience lookup methods (`getByCustomer`, `getByReviewedCustomer`, `getByReviewerAndReviewed`).  
* **Design patterns / frameworks**  
  * Spring `@Service` + constructor injection (`@Inject`).  
  * Repository pattern (`CustomerReviewRepository`).  
  * Generic service base (`SalesManagerEntityServiceImpl`) that probably supplies CRUD methods.  
  * Apache Commons `Validate` for defensive argument checking.  

---

## 2. Detailed Description  

### Core Components  
| Component | Role |
|-----------|------|
| `CustomerReviewRepository` | JPA repository that exposes custom query methods for reviews. |
| `CustomerService` | Used to fetch the reviewed `Customer` and to persist its updated aggregate data. |
| `SalesManagerEntityServiceImpl` | Generic CRUD implementation that the service extends. |
| `CustomerReviewServiceImpl` | Orchestrates review persistence and aggregate rating updates. |

### Execution Flow  

1. **Create/Update**  
   * `create()` or `update()` receives a `CustomerReview`.  
   * Delegates to private `saveOrUpdate()` which:  
     * Validates the review and its linked customers.  
     * Reloads the *reviewed* customer to ensure we work with the most recent data.  
     * Calculates the new total rating and average rating (using `BigDecimal`).  
     * Persists the review (`super.save(review)`), updates the customer’s rating fields (`customerService.update(customer)`), and finally sets the updated customer back on the review object.  

2. **Lookup**  
   * `getByCustomer()` – retrieves reviews where the supplied customer is the *reviewer*.  
   * `getByReviewedCustomer()` – retrieves reviews where the supplied customer is the *reviewed* one.  
   * `getByReviewerAndReviewed()` – fetches the single review that links the two customers.  

3. **Cleanup** – None explicitly; relies on container‑managed transaction boundaries and repository lifecycle.  

### Assumptions & Constraints  
* The repository implements the methods used (`findByReviewer`, `findByReviewed`, `findByRevieweAndReviewed`).  
* The `Customer` entity contains `customerReviewAvg` (BigDecimal) and `customerReviewCount` (Integer) fields that are meant to be kept in sync.  
* All operations are expected to be executed within a single transaction (implicit via Spring).  
* The service assumes that the `CustomerReview` passed to `saveOrUpdate` has non‑null `customer` and `reviewedCustomer`.  

### Architecture Choices  
* **Explicit validation**: Using `Validate` ensures early failure on bad input.  
* **Aggregate update logic**: Placed in the service layer to keep entities thin.  
* **Transactional safety**: Not declared explicitly; the class should be annotated with `@Transactional` or rely on a surrounding transactional boundary.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `saveOrUpdate(CustomerReview review)` | Core helper that persists a review and updates the reviewed customer’s rating aggregates. | `CustomerReview review` | void | Validates, recalculates aggregates, persists review and customer. |
| `create(CustomerReview review)` | Public API for creating a new review. | `CustomerReview review` | void | Delegates to `saveOrUpdate`. |
| `update(CustomerReview review)` | Public API for updating an existing review. | `CustomerReview review` | void | Delegates to `saveOrUpdate`. |
| `getByCustomer(Customer customer)` | Returns all reviews written by the supplied customer. | `Customer customer` | `List<CustomerReview>` | None. |
| `getByReviewedCustomer(Customer customer)` | Returns all reviews written *about* the supplied customer. | `Customer customer` | `List<CustomerReview>` | None. |
| `getByReviewerAndReviewed(Long reviewer, Long reviewed)` | Returns a single review that matches both reviewer and reviewed IDs. | `Long reviewer`, `Long reviewed` | `CustomerReview` | None. |

**Reusable utilities** – The service relies on `Validate` for defensive checks; no custom utilities defined here.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| Spring Framework (`@Service`, `@Inject`) | Framework | Core DI and component scanning. |
| Apache Commons Lang (`Validate`) | Third‑party | Simple input validation. |
| JPA / Spring Data (`CustomerReviewRepository`) | Framework | Repository pattern for persistence. |
| `SalesManagerEntityServiceImpl` | Project‑specific | Generic CRUD base; likely pulls in Spring Data or JPA. |
| `CustomerService` | Project‑specific | Service for managing `Customer` entities. |
| `java.math.BigDecimal` | JDK | Precise numeric operations. |
| `java.util.List` | JDK | Standard collections. |

All dependencies are either standard Java/JDK, well‑known libraries, or internal to the SalesManager codebase.

---

## 5. Additional Notes  

### Potential Issues  
1. **Typo in repository method** – `findByRevieweAndReviewed` should probably be `findByReviewerAndReviewed`. If the method name is misspelled, Spring Data will fail to generate the query, causing a runtime exception.  
2. **Precision loss** – The calculation of `avg` converts the `BigDecimal` result to a `double` before creating a new `BigDecimal`. This introduces rounding errors. It is better to keep everything in `BigDecimal` (e.g., `averageRating = totalRating.divide(BigDecimal.valueOf(count), RoundingMode.HALF_UP);`).  
3. **Concurrency / race conditions** – Two concurrent review creations for the same customer could read the same `count` and `averageRating`, leading to lost updates. Optimistic locking on the `Customer` entity (e.g., a `@Version` field) or a database‑level lock should be considered.  
4. **Null check for review rating** – The code assumes `review.getReviewRating()` is non‑null. A guard should be added to avoid `NullPointerException`.  
5. **Transaction boundaries** – The class is not annotated with `@Transactional`. While Spring may manage transactions at the repository level, the update of both review and customer should be atomic. Adding `@Transactional` (either on the class or the public methods) would guarantee this.  
6. **Redundant `review.setReviewedCustomer(customer)`** – This occurs *after* `super.save(review)`, meaning the persisted review will still reference the stale customer object. The customer should be set before saving the review.

### Recommendations  
- **Fix the repository method name** and verify all query method names match the entity fields.  
- **Refactor the rating calculation** to stay entirely in `BigDecimal` and apply proper rounding.  
- **Add optimistic locking** to the `Customer` entity or use a synchronized block / database lock.  
- **Guard against null review ratings** and possibly validate that the rating is within an expected range (e.g., 1–5).  
- **Annotate the class or methods with `@Transactional`** to ensure atomicity of review & customer updates.  
- **Move `review.setReviewedCustomer(customer)` before calling `super.save(review)`**.  
- **Unit tests**: Add tests for create/update flows, especially under concurrent scenarios, to ensure rating aggregates are computed correctly.  

By addressing these points, the service will become more robust, maintainable, and less prone to subtle bugs.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.customer.review;

import java.math.BigDecimal;
import java.util.List;

import javax.inject.Inject;

import org.apache.commons.lang3.Validate;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.customer.review.CustomerReviewRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.business.services.customer.CustomerService;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.customer.review.CustomerReview;

@Service("customerReviewService")
public class CustomerReviewServiceImpl extends
	SalesManagerEntityServiceImpl<Long, CustomerReview> implements CustomerReviewService {
	
	private CustomerReviewRepository customerReviewRepository;
	
	@Inject
	private CustomerService customerService;
	
	@Inject
	public CustomerReviewServiceImpl(
			CustomerReviewRepository customerReviewRepository) {
			super(customerReviewRepository);
			this.customerReviewRepository = customerReviewRepository;
	}
	
	
	private void saveOrUpdate(CustomerReview review) throws ServiceException {
		

		Validate.notNull(review,"CustomerReview cannot be null");
		Validate.notNull(review.getCustomer(),"CustomerReview.customer cannot be null");
		Validate.notNull(review.getReviewedCustomer(),"CustomerReview.reviewedCustomer cannot be null");
		
		
		//refresh customer
		Customer customer = customerService.getById(review.getReviewedCustomer().getId());
		
		//ajust product rating
		Integer count = 0;
		if(customer.getCustomerReviewCount()!=null) {
			count = customer.getCustomerReviewCount();
		}
				
		
		

		BigDecimal averageRating = customer.getCustomerReviewAvg();
		if(averageRating==null) {
			averageRating = new BigDecimal(0);
		}
		//get reviews

		
		BigDecimal totalRating = averageRating.multiply(new BigDecimal(count));
		totalRating = totalRating.add(new BigDecimal(review.getReviewRating()));
		
		count = count + 1;
		double avg = totalRating.doubleValue() / count;
		
		customer.setCustomerReviewAvg(new BigDecimal(avg));
		customer.setCustomerReviewCount(count);
		super.save(review);
		
		customerService.update(customer);
		
		review.setReviewedCustomer(customer);

		
	}
	
	public void update(CustomerReview review) throws ServiceException {
		this.saveOrUpdate(review);
	}
	
	public void create(CustomerReview review) throws ServiceException {
		this.saveOrUpdate(review);
	}
	
	

	@Override
	public List<CustomerReview> getByCustomer(Customer customer) {
		Validate.notNull(customer,"Customer cannot be null");
		return customerReviewRepository.findByReviewer(customer.getId());
	}

	@Override
	public List<CustomerReview> getByReviewedCustomer(Customer customer) {
		Validate.notNull(customer,"Customer cannot be null");
		return customerReviewRepository.findByReviewed(customer.getId());
	}


	@Override
	public CustomerReview getByReviewerAndReviewed(Long reviewer, Long reviewed) {
		Validate.notNull(reviewer,"Reviewer customer cannot be null");
		Validate.notNull(reviewed,"Reviewer customer cannot be null");
		return customerReviewRepository.findByRevieweAndReviewed(reviewer, reviewed);
	}

}



```
