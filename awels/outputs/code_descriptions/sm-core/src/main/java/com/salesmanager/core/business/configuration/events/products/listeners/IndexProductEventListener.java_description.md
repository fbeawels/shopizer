# IndexProductEventListener.java

## Review

## 1. Summary

The **`IndexProductEventListener`** is a Spring component that listens for product‑related events (`ProductEvent` and its subclasses) and synchronises the search index accordingly.  
Key responsibilities:

| Component | Role |
|-----------|------|
| `SearchService` | Indexes or deletes documents in the search backend. |
| `ProductService` | Loads the full product from the database when the event only contains a partial reference. |
| Event handling methods (`saveProduct`, `deleteProduct`, etc.) | Update the index when a product, variant, image or attribute is created, updated or removed. |

Design patterns & frameworks:
* Spring’s **ApplicationListener** interface for event handling.
* Dependency injection (`@Autowired`).
* Conditional execution (`@Value("${search.noindex:false}")`).

The listener is intentionally lightweight and stateless; it just translates event data into search‑index operations.

---

## 2. Detailed Description

### Execution Flow

1. **Event Arrival** – A `ProductEvent` (or any subclass) is published in the Spring context.  
2. **Listener Triggered** – `onApplicationEvent(ProductEvent event)` is invoked.  
3. **No‑Index Check** – If the `search.noindex` flag is set, the method exits immediately.  
4. **Dispatch to Specific Handler** – Using `instanceof` checks, the event is routed to the appropriate private method (e.g., `saveProduct`, `deleteProductVariant`, …).  
5. **Refresh Product** – Each handler calls `productOfEvent` which:
   * Pulls the product’s ID from the event.
   * Uses `productService.findOne(id, store)` to obtain the fully‑populated entity.
   * Returns the refreshed product (or throws a runtime exception if anything goes wrong).
6. **Index Manipulation** – The handler either:
   * Calls `searchService.index(store, product)` to create/update a document.
   * Calls `searchService.deleteDocument(store, product)` to remove a document.
7. **Post‑processing** – All methods return `void`; any exception is wrapped in a `RuntimeException` and propagated.

### Assumptions & Constraints

* The event’s `Product` reference is non‑null and contains at least the ID and merchant store.  
* All product modifications (variants, images, attributes) are represented by their own events that include the modified entity.  
* The `SearchService` performs the heavy lifting; the listener does not concern itself with indexing specifics.  
* The listener is a singleton; it does not maintain any mutable state beyond what is supplied by the events.  

### Architectural Observations

* **Coupling** – The listener is tightly coupled to the concrete event subclasses via `instanceof`. Adding a new event type requires editing this class.  
* **Duplication** – The logic for adding/removing variants, images, attributes is almost identical; this can be refactored into a generic helper.  
* **Error Handling** – All checked exceptions are swallowed and re‑thrown as `RuntimeException`. This can mask the original cause and make debugging harder.  
* **Logging** – The class prints to `System.out` for an error case; in a production Spring app this should use a proper logger.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `onApplicationEvent(ProductEvent)` | Entry point for all product events; dispatches to specific handlers. | `ProductEvent` | `void` | Calls search service via helper methods. |
| `productOfEvent(ProductEvent)` | Refreshes a product from the DB to obtain a fully‑populated entity. | `ProductEvent` | `Product` | Throws `RuntimeException` on failure; logs to console. |
| `saveProduct(SaveProductEvent)` | Indexes a product after creation or update. | `SaveProductEvent` | `void` | Calls `searchService.index`. |
| `deleteProduct(DeleteProductEvent)` | Removes a product from the index. | `DeleteProductEvent` | `void` | Calls `searchService.deleteDocument`. |
| `saveProductVariant(SaveProductVariantEvent)` | Adds/updates a variant in the product and re‑indexes. | `SaveProductVariantEvent` | `void` | Mutates product’s variant set; indexes. |
| `deleteProductVariant(DeleteProductVariantEvent)` | Removes a variant and re‑indexes. | `DeleteProductVariantEvent` | `void` | Mutates product’s variant set; indexes. |
| `saveProductImage(SaveProductImageEvent)` | Adds an image to the product and re‑indexes. | `SaveProductImageEvent` | `void` | Mutates product’s image set; indexes. |
| `deleteProductImage(DeleteProductImageEvent)` | No operation – image removal does not trigger a re‑index. | `DeleteProductImageEvent` | `void` | None. |
| `saveProductAttribute(SaveProductAttributeEvent)` | Adds an attribute and re‑indexes. | `SaveProductAttributeEvent` | `void` | Mutates product’s attribute set; indexes. |
| `deleteProductAttribute(DeleteProductAttributeEvent)` | Removes an attribute and re‑indexes. | `DeleteProductAttributeEvent` | `void` | Mutates product’s attribute set; indexes. |

All helper methods are package‑private (no access modifier) and rely on Spring’s component wiring.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.context.ApplicationListener` | Spring core | Event listener interface. |
| `org.springframework.beans.factory.annotation.Autowired` / `@Value` | Spring core | Field injection. |
| `com.salesmanager.core.business.services.search.SearchService` | Internal | Interface for search index operations. |
| `com.salesmanager.core.business.services.catalog.product.ProductService` | Internal | Service to fetch full product details. |
| `com.salesmanager.core.business.exception.ServiceException` | Internal | Checked exception thrown by `SearchService`. |
| `com.salesmanager.core.model.*` | Internal | Domain models (`Product`, `ProductVariant`, etc.). |
| Java Streams, Collections | JDK | For filtering and constructing sets. |

No external third‑party libraries are used beyond Spring and the internal `salesmanager` codebase.

---

## 5. Additional Notes & Recommendations

### 5.1. Code Quality / Style

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **System.out.println** for error logging | Misses application logs; non‑structured | Replace with SLF4J logger (`private static final Logger log = LoggerFactory.getLogger(...)`). |
| **Generic `RuntimeException` wrapping** | Hides original exception type, making stack traces harder to interpret | Use a dedicated unchecked exception (`SearchEventProcessingException`) or propagate the original checked exception. |
| **Field injection** (`@Autowired`) | Harder to test, less clear dependencies | Switch to constructor injection (`public IndexProductEventListener(SearchService searchService, ProductService productService, @Value("${search.noindex:false}") boolean noIndex)`). |
| **`instanceof` dispatch** | Fragile; new event types require modification | Use a map of `Class<? extends ProductEvent>` → handler lambda or interface (e.g., a strategy pattern). |
| **Duplicate logic** (add/remove variants, images, attributes) | Maintenance burden | Extract a generic helper that accepts a `Function<Product, Set<T>>` to get the collection and a `Consumer<Product, T>` to set it back. |
| **Potential NPEs** (e.g., `variant.getId()` if `variant` null) | Runtime failures | Validate non‑null before usage; throw meaningful exception. |
| **Hard‑coded `boolean noIndex` default** | Mis‑configuration risk | Document property usage; consider `@ConfigurationProperties`. |
| **Empty method** `deleteProductImage` | Confusing; comment indicates intentional skip | Either remove the method entirely or document the rationale. |
| **Method visibility** (`void` with no modifier) | Defaults to package‑private; may be unintentionally exposed | Explicitly mark as `private` or `protected`. |

### 5.2. Performance & Scalability

* **Re‑indexing after each variant/image/attribute change** may be heavy if events arrive in rapid succession. Consider batching changes per product or using an “in‑memory delta” strategy.  
* **Fetching the full product for every event** can add overhead. If the event already contains all needed fields, skip the DB call.  
* **Using Sets for variants, images, attributes** ensures uniqueness but incurs allocation on each event. Reuse existing sets where possible.

### 5.3. Error Handling & Recovery

* Currently any exception aborts the entire listener chain by propagating a `RuntimeException`.  
* Consider using Spring’s `@EventListener` with `@Async` to avoid blocking the publisher.  
* Add retry logic for transient failures in `SearchService` (e.g., via Spring Retry).

### 5.4. Future Enhancements

1. **Event Coalescing** – Merge multiple events for the same product into a single re‑index operation.  
2. **Granular Indexing** – Allow selective indexing of only affected attributes/variants instead of full product.  
3. **Health Checks** – Expose a health endpoint that verifies search index consistency.  
4. **Metrics** – Instrument the listener with Micrometer to expose processing counts and latencies.  
5. **Unit Tests** – Provide comprehensive tests for each event path, including error scenarios.

---

### Bottom Line

The listener fulfills its core purpose—keeping the search index in sync with product changes—but several improvements could make it more robust, maintainable, and performant. Addressing the highlighted issues will reduce fragility, improve observability, and ease future evolution.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.configuration.events.products.listeners;

import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

import com.salesmanager.core.business.configuration.events.products.DeleteProductAttributeEvent;
import com.salesmanager.core.business.configuration.events.products.DeleteProductEvent;
import com.salesmanager.core.business.configuration.events.products.DeleteProductImageEvent;
import com.salesmanager.core.business.configuration.events.products.DeleteProductVariantEvent;
import com.salesmanager.core.business.configuration.events.products.ProductEvent;
import com.salesmanager.core.business.configuration.events.products.SaveProductAttributeEvent;
import com.salesmanager.core.business.configuration.events.products.SaveProductEvent;
import com.salesmanager.core.business.configuration.events.products.SaveProductImageEvent;
import com.salesmanager.core.business.configuration.events.products.SaveProductVariantEvent;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.search.SearchService;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.merchant.MerchantStore;

/**
 * Index product in search module if it is configured to do so !
 * 
 * Should receive events that a product was created or updated or deleted
 * 
 * @author carlsamson
 *
 */
@Component
public class IndexProductEventListener implements ApplicationListener<ProductEvent> {

	@Autowired
	private SearchService searchService;

	@Autowired
	private ProductService productService;
	
    @Value("${search.noindex:false}")//skip indexing process
    private boolean noIndex;

	/**
	 * Listens to ProductEvent and ProductVariantEvent
	 */
	@Override
	public void onApplicationEvent(ProductEvent event) {
		
		
		if(!noIndex) {

			if (event instanceof SaveProductEvent) {
				saveProduct((SaveProductEvent) event);
			}
	
			if (event instanceof DeleteProductEvent) {
				deleteProduct((DeleteProductEvent) event);
			}
	
			if (event instanceof SaveProductVariantEvent) {
				saveProductVariant((SaveProductVariantEvent) event);
			}
	
			if (event instanceof DeleteProductVariantEvent) {
				deleteProductVariant((DeleteProductVariantEvent) event);
			}
			
			if (event instanceof SaveProductImageEvent) {
				saveProductImage((SaveProductImageEvent) event);
			}
	
			if (event instanceof DeleteProductImageEvent) {
				deleteProductImage((DeleteProductImageEvent) event);
			}
			
			if (event instanceof SaveProductAttributeEvent) {
				saveProductAttribute((SaveProductAttributeEvent) event);
			}
	
			if (event instanceof DeleteProductAttributeEvent) {
				deleteProductAttribute((DeleteProductAttributeEvent) event);
			}
			
			
		
		}

	}
	
	private Product productOfEvent(ProductEvent event) {
		
		Product product = event.getProduct();
		MerchantStore store = product.getMerchantStore();
		try {

			/**
			 * Refresh product
			 */

			Product fullProduct = productService.findOne(product.getId(), store);
			
			if(fullProduct != null) {
				product = fullProduct;
			} else {
				System.out.println("Product not loaded");
			}
			
		return product;
			
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
		
	}

	void saveProduct(SaveProductEvent event) {
		
		try {
			Product product = productOfEvent(event);
			MerchantStore store = product.getMerchantStore();
	
			searchService.index(store, product);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	void deleteProduct(DeleteProductEvent event) {

		Product product = event.getProduct();
		MerchantStore store = product.getMerchantStore();

		try {
			searchService.deleteDocument(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}
	}

	void saveProductVariant(SaveProductVariantEvent event) {

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();

		ProductVariant variant = event.getVariant();// to be removed

		/**
		 * add new variant to be saved
		 **/

		List<ProductVariant> filteredVariants = product.getVariants().stream()
				.filter(i -> variant.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());

		filteredVariants.add(variant);

		Set<ProductVariant> allVariants = new HashSet<ProductVariant>(filteredVariants);
		product.setVariants(allVariants);

		try {
			searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}

	}

	void deleteProductVariant(DeleteProductVariantEvent event) {

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();

		ProductVariant variant = event.getVariant();// to be removed

		/**
		 * remove variant to be saved
		 **/

		List<ProductVariant> filteredVariants = product.getVariants().stream()
				.filter(i -> variant.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());

		Set<ProductVariant> allVariants = new HashSet<ProductVariant>(filteredVariants);
		product.setVariants(allVariants);

		try {
			searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}

	}
	

	void saveProductImage(SaveProductImageEvent event) {

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();


		ProductImage image = event.getProductImage();// to be removed

		/**
		 * add new image to be saved
		 **/

		List<ProductImage> filteredImages = product.getImages().stream()
				.filter(i -> i.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());

		filteredImages.add(image);

		Set<ProductImage> allInmages = new HashSet<ProductImage>(filteredImages);
		product.setImages(allInmages);

		try {
			searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}

	}
	
	void deleteProductImage(DeleteProductImageEvent event) {
		
		//Product will be updated anyway so there is no need to reindex following an image removal
		return;
		
		/**

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();

		List<ProductImage> filteredImages = product.getImages().stream()
				.filter(i -> i.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());


		Set<ProductImage> allImages = new HashSet<ProductImage>(filteredImages);
		product.setImages(allImages);

		try {
			

			//searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}
		**/

	}
	
	void saveProductAttribute(SaveProductAttributeEvent event) {

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();

		ProductAttribute attribute = event.getProductAttribute();// to be removed

		/**
		 * add new attribute to be saved
		 **/

		List<ProductAttribute> filteredAttributes = product.getAttributes().stream()
				.filter(i -> i.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());

		filteredAttributes.add(attribute);

		Set<ProductAttribute> allAttributes = new HashSet<ProductAttribute>(filteredAttributes);
		product.setAttributes(allAttributes);

		try {
			searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}

	}
	
	void deleteProductAttribute(DeleteProductAttributeEvent event) {

		Product product = productOfEvent(event);

		MerchantStore store = product.getMerchantStore();

		/**
		 * add new attribute to be saved
		 **/

		List<ProductAttribute> filteredAttributes = product.getAttributes().stream()
				.filter(i -> i.getId().longValue() != i.getId().longValue()).collect(Collectors.toList());

		Set<ProductAttribute> allAttributes = new HashSet<ProductAttribute>(filteredAttributes);
		product.setAttributes(allAttributes);

		try {
			searchService.index(store, product);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}

	}

	/**
	 * Get document by product id and document exist if event is Product delete
	 * document create document if event is Variant get document get variants
	 * replace variant
	 */

}



```
