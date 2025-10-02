# ProductReviewServiceImpl.java

## Review

## 1. Summary
**Purpose**  
`ProductReviewServiceImpl` is a Spring service that manages product reviews. It provides CRUD‑like operations for reviews, query methods that filter by customer, product, and language, and it automatically updates a product’s average rating and review count whenever a review is created or updated.

**Key components**
| Component | Role |
|-----------|------|
| `ProductReviewRepository` | Spring‑Data repository for `ProductReview` entities (used for queries). |
| `ProductService` | Service used to refresh the `Product` entity and persist the updated rating/​count. |
| `SalesManagerEntityServiceImpl<Long, ProductReview>` | Base service providing generic CRUD methods (`save`, `findById`, etc.). |
| `ProductReviewService` | Service interface that declares the public API. |

**Notable patterns/frameworks**  
* Spring’s `@Service` and constructor injection (`@Inject`).  
* Apache Commons `Validate` for argument validation.  
* Generic DAO pattern via `SalesManagerEntityServiceImpl`.  
* The service relies on Spring Data JPA/Hibernate under the hood.

---

## 2. Detailed Description
1. **Construction & Wiring**  
   * The constructor receives a `ProductReviewRepository` and passes it to the parent class.  
   * `productService` is injected via field injection – a mix of constructor and field injection that could be unified for consistency.

2. **Query methods**  
   * `getByCustomer`, `getByProduct`, `getByProductAndCustomer`, `getByProduct(Product, Language)`, and `getByProductNoCustomers` delegate directly to the repository.  
   * These methods allow clients to retrieve reviews based on various filters.

3. **Persisting a review (`saveOrUpdate`)**  
   * Validates that the review, its product, and its customer are non‑null.  
   * Reloads the product from the database (`productService.getById`) to avoid stale state.  
   * Calculates the new average rating and count:
     * Reads current `productReviewAvg` and `productReviewCount`.  
     * Computes the total rating (`avg * count`) and adds the new review’s rating.  
     * Increments the count (even on an update).  
     * Stores the new average (converted to `BigDecimal`).  
   * Persists the review (`super.save(review)`) and updates the product (`productService.update(product)`).

4. **Public create/update**  
   * Both `create` and `update` simply delegate to `saveOrUpdate`.  
   * No distinction is made between inserting a new row and updating an existing one – the repository handles that, but the rating logic treats every call as an addition, which is a bug when updating.

5. **Transactional context**  
   * The class itself isn’t annotated with `@Transactional`; it relies on the base service or callers to provide a transaction.  
   * Without an explicit transaction, the two database writes (review and product) could be inconsistent if a failure occurs between them.

6. **Cleanup**  
   * No explicit cleanup; the service is stateless aside from the injected beans.

**Assumptions & Constraints**  
* The repository methods (`findBy…`) return the expected lists; they are assumed to be defined elsewhere.  
* The `review.getReviewRating()` is an integer (or numeric) value without validation of bounds.  
* No handling of review deletions or rating recalculations when a review is removed.  
* The code expects a single database transaction for both writes.

**Overall architecture**  
A thin service layer that delegates most persistence work to a generic base class and a repository, while adding domain‑specific logic (rating aggregation). The design follows common Spring + JPA patterns but mixes injection styles and lacks transaction safety for complex operations.

---

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `getByCustomer(Customer)` | Fetch all reviews for a customer. | `Customer` | `List<ProductReview>` | No DB writes. |
| `getByProduct(Product)` | Fetch all reviews for a product. | `Product` | `List<ProductReview>` | No DB writes. |
| `getByProductAndCustomer(Long, Long)` | Fetch a single review by product and customer. | `productId`, `customerId` | `ProductReview` | No DB writes. |
| `getByProduct(Product, Language)` | Fetch reviews for a product in a specific language. | `Product`, `Language` | `List<ProductReview>` | No DB writes. |
| `getByProductNoCustomers(Product)` | Fetch reviews for a product that have no customer linked (likely a join‑filter). | `Product` | `List<ProductReview>` | No DB writes. |
| `saveOrUpdate(ProductReview)` *(private)* | Validates input, updates product rating/​count, persists review, updates product. | `ProductReview` | None | DB writes to `ProductReview` and `Product`. |
| `update(ProductReview)` | Public API to update a review (calls `saveOrUpdate`). | `ProductReview` | None | Same as `saveOrUpdate`. |
| `create(ProductReview)` | Public API to create a review (calls `saveOrUpdate`). | `ProductReview` | None | Same as `saveOrUpdate`. |

**Reusable / Utility methods**  
The only “utility” logic is inside `saveOrUpdate`; it could be extracted into a dedicated rating helper or service to keep the controller cleaner.

---

## 4. Dependencies
| Library / Framework | Use | Type |
|---------------------|-----|------|
| Spring Framework (`@Service`, `@Inject`) | Dependency injection, service stereotype | Third‑party |
| Apache Commons Lang (`Validate`) | Parameter validation | Third‑party |
| JPA / Hibernate (via `SalesManagerEntityServiceImpl`) | ORM persistence | Third‑party |
| `ProductReviewRepository` | Repository interface for CRUD & custom queries | Project |
| `ProductService` | Business logic for products | Project |
| Java Standard Library | `BigDecimal`, collections | Standard |

No platform‑specific dependencies are evident; the code is portable across Java EE/Spring containers.

---

## 5. Additional Notes
### Edge‑cases & potential bugs
1. **Updating an existing review** – The method increments the review count regardless of whether the review is new or updated. This will inflate the average rating and count when a review is edited.
2. **Concurrent reviews** – Two simultaneous reviews will read the same `count` and `average`, leading to lost updates. No optimistic locking or transactional isolation is enforced.
3. **Rating bounds** – No validation of `review.getReviewRating()` (e.g., ensuring it is between 1 and 5). A rogue value could corrupt the average.
4. **Null rating** – If `review.getReviewRating()` is null, the code will throw a `NullPointerException` when wrapped in `BigDecimal`.
5. **Floating‑point precision** – The average is calculated using `double` and then wrapped back into `BigDecimal`. This can introduce rounding errors. It would be safer to keep the entire computation in `BigDecimal`.
6. **Transaction handling** – Without an explicit `@Transactional` annotation, the two separate writes (review and product) may not be atomic. A failure after saving the review but before updating the product would leave data in an inconsistent state.
7. **No delete operation** – The service does not provide a method to delete a review or to recalculate ratings after deletion.
8. **Naming inconsistency** – `getByProductNoCustomers` reads like it returns reviews *without* customers, but the repository method name `findByProductNoCustomers` may be misleading. Clarifying the intent would help maintainability.

### Suggested Improvements
| Area | Recommendation |
|------|----------------|
| **Method separation** | Split `saveOrUpdate` into `createReview` and `updateReview` that handle rating logic appropriately. |
| **Rating calculation** | Use `BigDecimal` for all math, avoid `double`. Validate rating bounds. |
| **Concurrency** | Apply optimistic locking on `Product` (e.g., version field) or use database atomic updates. |
| **Transaction safety** | Annotate service methods with `@Transactional` and consider propagation settings. |
| **Dependency injection** | Use constructor injection consistently for all collaborators. |
| **Deletion support** | Add `delete` and rating recomputation logic. |
| **Documentation** | Javadoc for public methods, especially clarifying the effect on average/count. |
| **Error handling** | Wrap repository exceptions into `ServiceException` with meaningful messages. |
| **Unit tests** | Test creation, update, concurrent scenarios, and boundary ratings. |

### Future Enhancements
* **Rating aggregation optimization** – Store pre‑computed totals in `Product` to avoid recomputation on each review.  
* **Event‑driven architecture** – Publish events on review creation/update and let a listener recalculate ratings.  
* **Caching** – Cache product rating data to reduce database load for read‑heavy scenarios.  
* **Internationalization** – Extend rating logic to support locale‑specific rating scales.

Overall, the class provides a solid foundation for review management but requires attention to correctness in the rating logic, transaction safety, and concurrency handling before it can be safely deployed in a production environment.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.product.review;

import java.math.BigDecimal;
import java.util.List;

import javax.inject.Inject;

import org.apache.commons.lang3.Validate;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.catalog.product.review.ProductReviewRepository;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.review.ProductReview;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.reference.language.Language;

@Service("productReviewService")
public class ProductReviewServiceImpl extends
		SalesManagerEntityServiceImpl<Long, ProductReview> implements
		ProductReviewService {


	private ProductReviewRepository productReviewRepository;
	
	@Inject
	private ProductService productService;
	
	@Inject
	public ProductReviewServiceImpl(
			ProductReviewRepository productReviewRepository) {
			super(productReviewRepository);
			this.productReviewRepository = productReviewRepository;
	}

	@Override
	public List<ProductReview> getByCustomer(Customer customer) {
		return productReviewRepository.findByCustomer(customer.getId());
	}

	@Override
	public List<ProductReview> getByProduct(Product product) {
		return productReviewRepository.findByProduct(product.getId());
	}
	
	@Override
	public ProductReview getByProductAndCustomer(Long productId, Long customerId) {
		return productReviewRepository.findByProductAndCustomer(productId, customerId);
	}
	
	@Override
	public List<ProductReview> getByProduct(Product product, Language language) {
		return productReviewRepository.findByProduct(product.getId(), language.getId());
	}
	
	private void saveOrUpdate(ProductReview review) throws ServiceException {
		

		Validate.notNull(review,"ProductReview cannot be null");
		Validate.notNull(review.getProduct(),"ProductReview.product cannot be null");
		Validate.notNull(review.getCustomer(),"ProductReview.customer cannot be null");
		
		
		//refresh product
		Product product = productService.getById(review.getProduct().getId());
		
		//ajust product rating
		Integer count = 0;
		if(product.getProductReviewCount()!=null) {
			count = product.getProductReviewCount();
		}
				
		
		

		BigDecimal averageRating = product.getProductReviewAvg();
		if(averageRating==null) {
			averageRating = new BigDecimal(0);
		}
		//get reviews

		
		BigDecimal totalRating = averageRating.multiply(new BigDecimal(count));
		totalRating = totalRating.add(new BigDecimal(review.getReviewRating()));
		
		count = count + 1;
		double avg = totalRating.doubleValue() / count;
		
		product.setProductReviewAvg(new BigDecimal(avg));
		product.setProductReviewCount(count);
		super.save(review);
		
		productService.update(product);
		
		review.setProduct(product);
		
	}
	
	public void update(ProductReview review) throws ServiceException {
		this.saveOrUpdate(review);
	}
	
	public void create(ProductReview review) throws ServiceException {
		this.saveOrUpdate(review);
	}

	/* (non-Javadoc)
	 * @see com.salesmanager.core.business.services.catalog.product.review.ProductReviewService#getByProductNoObjects(com.salesmanager.core.model.catalog.product.Product)
	 */
	@Override
	public List<ProductReview> getByProductNoCustomers(Product product) {
		return productReviewRepository.findByProductNoCustomers(product.getId());
	}


}



```
