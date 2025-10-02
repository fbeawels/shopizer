# SearchFacadeImpl.java

## Review

## 1. Summary
`SearchFacadeImpl` is a Spring‐managed service that bridges the front‑end search API to the underlying search engine (ElasticSearch or a similar service).  
* **Key responsibilities**
  * Index all catalog data (`indexAllData`).
  * Execute a normal search request (`search` → private `search`).
  * Convert raw domain objects (`Product`, `Category`) into readable DTOs (`ReadableProduct`, `ReadableCategory`).
  * Provide autocomplete suggestions (`autocompleteRequest`).
* **Core collaborators**
  * `SearchService` – low‑level query & indexing logic.
  * `ProductService`, `CategoryService` – fetch catalog entities.
  * `PricingService` – enrich products with pricing.
  * `ImageFilePath` – build URLs for product images.
* **Design patterns / frameworks**
  * Spring’s `@Service`, dependency injection (`@Inject`, `@Qualifier`), and asynchronous execution (`@Async`).
  * Use of helper utilities (`ReadableProductPopulator`, `ReadableCategoryPopulator`) to convert entities to DTOs.
  * The `SearchRequest`/`SearchResponse` pattern, resembling a request/response DTO for the search engine.

---

## 2. Detailed Description
1. **Initialization**  
   * Spring creates an instance of `SearchFacadeImpl` and injects all required services.  
   * The `ImageFilePath` bean is referenced with `@Qualifier("img")`, indicating a named bean configuration elsewhere.

2. **Runtime behavior**  
   * **Indexing** – `indexAllData` runs asynchronously. It retrieves all products for a store and calls `searchService.index` for each one. Any `ServiceException` inside the stream is wrapped into a `RuntimeException`.  
   * **Searching** – The public `search` method delegates to a private overload that builds a `SearchRequest`, validates parameters, and forwards the call to `searchService.search`. The result (`SearchResponse`) is unwrapped into a list of `SearchItem`.  
   * **Autocomplete** – Similar to searching, but it calls `searchService.searchKeywords` and extracts suggestion strings into a `ValueList`.  
   * **DTO conversion** – `convertProductToReadableProduct` and `convertCategoryToReadableCategory` use dedicated populator classes and handle conversion errors by throwing runtime wrappers.

3. **Cleanup** – None. The class is stateless aside from injected services.

4. **Assumptions / Constraints**  
   * `SearchService` is thread‑safe because `indexAllData` uses `@Async`.  
   * All DTO conversion logic is encapsulated in populator classes; no manual field mapping here.  
   * The `SearchRequest` object must be populated with store code, language, and search string only.  
   * `getCategoryFacets` is currently a stub (returns `null`), indicating unfinished feature or omitted logic.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects / Notes |
|--------|---------|--------|---------|----------------------|
| `indexAllData(MerchantStore store)` | Asynchronously index all products of a store. | `MerchantStore` | `void` | Runs asynchronously; throws unchecked `RuntimeException` if indexing fails. |
| `search(MerchantStore store, Language language, SearchProductRequest searchRequest)` | Public façade that returns a list of `SearchItem` objects. | Store, language, request DTO | `List<SearchItem>` | Delegates to private `search` method. |
| `private SearchResponse search(MerchantStore store, String languageCode, String query, Integer count, Integer start)` | Builds `SearchRequest`, validates, invokes `searchService`. | Same as above but flattened | `SearchResponse` | Wraps checked `ServiceException` into `ServiceRuntimeException`. |
| `private List<ReadableCategory> getCategoryFacets(...)` | Intended to translate facet data into readable categories. | Store, language, facets list | `List<ReadableCategory>` | Currently stubbed; returns `null`. |
| `private ReadableCategory convertCategoryToReadableCategory(...)` | Populates a `ReadableCategory` from a `Category` and facet count. | Store, language, count map, `Category` | `ReadableCategory` | Throws `ConversionRuntimeException` on failure. |
| `private ReadableProduct convertProductToReadableProduct(...)` | Populates a `ReadableProduct` from a `Product`. | Product, store, language | `ReadableProduct` | Uses `PricingService` and `ImageFilePath`. |
| `public ValueList autocompleteRequest(String word, MerchantStore store, Language language)` | Provides autocomplete suggestions. | Search keyword, store, language | `ValueList` (list of suggestion strings) | Wraps `ServiceException` into unchecked `RuntimeException`. |

*Reusable utilities:* `convertCategoryToReadableCategory` and `convertProductToReadableProduct` can be reused by other components that need the same DTO mapping.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.stereotype.Service` | Spring | Marks the class as a Spring service. |
| `org.springframework.beans.factory.annotation.Qualifier` | Spring | Injects a specific `ImageFilePath` bean. |
| `org.springframework.scheduling.annotation.Async` | Spring | Enables asynchronous execution. |
| `javax.inject.Inject` | CDI / Spring | Dependency injection. |
| `org.jsoup.helper.Validate` | Jsoup (third‑party) | Used for null checks; could be replaced with Apache Commons or Spring’s own `Assert`. |
| `org.slf4j.Logger` / `LoggerFactory` | SLF4J | Logging. |
| `com.salesmanager.core.business.services.*` | SalesManager core | Domain services. |
| `com.salesmanager.shop.populator.*` | SalesManager shop | DTO conversion utilities. |
| `modules.commons.search.*` | Custom / third‑party | Request/response DTOs for search. |
| `com.salesmanager.shop.utils.ImageFilePath` | SalesManager shop | Builds image URLs. |

All dependencies are either standard Spring components or project‑specific modules; no external cloud or platform SDKs.

---

## 5. Additional Notes & Recommendations

### 5.1. Incomplete Implementation
* `getCategoryFacets` is commented out and currently returns `null`. This will cause a `NullPointerException` wherever it’s used. If the method is no longer needed, it should be removed; otherwise, the implementation must be restored or a suitable placeholder added.

### 5.2. Error Handling
* The public methods wrap checked exceptions into unchecked runtime exceptions but lose the original stack trace context in some cases (e.g., `indexAllData` throws a raw `RuntimeException`). It would be clearer to use custom runtime wrappers (`ServiceRuntimeException`, `ConversionRuntimeException`) consistently.
* Consider logging the errors before rethrowing to aid troubleshooting.

### 5.3. Validation Library
* `org.jsoup.helper.Validate` is primarily meant for HTML parsing; using `org.springframework.util.Assert` or `org.apache.commons.lang3.Validate` would be more appropriate and clearer.

### 5.4. Asynchronous Configuration
* `@Async` requires `@EnableAsync` in a configuration class. Verify that this is present; otherwise, the method will execute synchronously.

### 5.5. Performance
* `indexAllData` streams over all products and calls `searchService.index` one by one. Depending on the volume, batching or bulk indexing might be more efficient.
* The autocomplete method requests a fixed count (`AUTOCOMPLETE_ENTRIES_COUNT`); consider making this configurable.

### 5.6. Thread Safety
* The service holds no mutable state, so it is thread‑safe. However, if `SearchService` or `ProductService` are not thread‑safe, additional safeguards may be needed.

### 5.7. Documentation & Naming
* Add JavaDoc to public methods explaining parameters and return values.
* Rename `search` (private) to something more descriptive like `executeSearch` to avoid confusion with the public overload.

### 5.8. Unit Tests
* The current code lacks unit tests. Mock `SearchService`, `ProductService`, etc., and verify:
  * Proper handling of null arguments.
  * Correct construction of `SearchRequest`.
  * Conversion logic in `convertProductToReadableProduct` and `convertCategoryToReadableCategory`.

### 5.9. Future Enhancements
* **Facet handling** – complete the `getCategoryFacets` logic or expose a public method that returns facets for UI rendering.
* **Pagination & Sorting** – add support for custom sort orders or filters.
* **Internationalization** – ensure that all strings in the service (e.g., log messages) respect locale settings.
* **Metrics** – integrate with Micrometer or similar to track search latency and indexing throughput.

---

**Overall Verdict**  
The class provides a clear façade for search-related operations and leverages Spring’s infrastructure effectively. However, it contains an unfinished method, inconsistent exception handling, and some minor design choices that could be tightened. Addressing the points above will make the component more robust, maintainable, and easier to test.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.search.facade;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import javax.inject.Inject;

import org.jsoup.helper.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ConversionException;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.category.CategoryService;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.search.SearchService;
import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.SearchProductRequest;
import com.salesmanager.shop.model.catalog.category.ReadableCategory;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.entity.ValueList;
import com.salesmanager.shop.populator.catalog.ReadableCategoryPopulator;
import com.salesmanager.shop.populator.catalog.ReadableProductPopulator;
import com.salesmanager.shop.store.api.exception.ConversionRuntimeException;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;
import com.salesmanager.shop.utils.ImageFilePath;

import modules.commons.search.request.Aggregation;
import modules.commons.search.request.SearchItem;
import modules.commons.search.request.SearchRequest;
import modules.commons.search.request.SearchResponse;

@Service("searchFacade")
public class SearchFacadeImpl implements SearchFacade {

	private static final Logger LOGGER = LoggerFactory.getLogger(SearchFacadeImpl.class);

	@Inject
	private SearchService searchService;

	@Inject
	private ProductService productService;

	@Inject
	private CategoryService categoryService;

	@Inject
	private PricingService pricingService;

	@Inject
	@Qualifier("img")
	private ImageFilePath imageUtils;

	private final static String CATEGORY_FACET_NAME = "categories";
	private final static String MANUFACTURER_FACET_NAME = "manufacturer";
	private final static int AUTOCOMPLETE_ENTRIES_COUNT = 15;

	/**
	 * Index all products from the catalogue Better stop the system, remove ES
	 * indexex manually restart ES and run this query
	 */
	@Override
	@Async
	public void indexAllData(MerchantStore store) throws Exception {
		List<Product> products = productService.listByStore(store);

		products.stream().forEach(p -> {
			try {
				searchService.index(store, p);
			} catch (ServiceException e) {
				throw new RuntimeException("Exception while indexing products", e);
			}
		});

	}

	@Override
	public List<SearchItem> search(MerchantStore store, Language language, SearchProductRequest searchRequest) {
		SearchResponse response = search(store, language.getCode(), searchRequest.getQuery(), searchRequest.getCount(),
				searchRequest.getStart());
		return response.getItems();
	}

	private SearchResponse search(MerchantStore store, String languageCode, String query, Integer count,
			Integer start) {
		
		Validate.notNull(query,"Search Keyword must not be null");
		Validate.notNull(languageCode, "Language cannot be null");
		Validate.notNull(store,"MerchantStore cannot be null");
		
		
		try {
			LOGGER.debug("Search " + query);
			SearchRequest searchRequest = new SearchRequest();
			searchRequest.setLanguage(languageCode);
			searchRequest.setSearchString(query);
			searchRequest.setStore(store.getCode().toLowerCase());
			
			
			//aggregations
			
			//TODO add scroll
			return searchService.search(store, languageCode, searchRequest, count, start);

		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}
	}

	private List<ReadableCategory> getCategoryFacets(MerchantStore merchantStore, Language language, List<Aggregation> facets) {
		
		
		/**
		List<SearchFacet> categoriesFacets = facets.entrySet().stream()
				.filter(e -> CATEGORY_FACET_NAME.equals(e.getKey())).findFirst().map(Entry::getValue)
				.orElse(Collections.emptyList());

		if (CollectionUtils.isNotEmpty(categoriesFacets)) {

			List<String> categoryCodes = categoriesFacets.stream().map(SearchFacet::getName)
					.collect(Collectors.toList());

			Map<String, Long> productCategoryCount = categoriesFacets.stream()
					.collect(Collectors.toMap(SearchFacet::getKey, SearchFacet::getCount));

			List<Category> categories = categoryService.listByCodes(merchantStore, categoryCodes, language);
			return categories.stream().map(category -> convertCategoryToReadableCategory(merchantStore, language,
					productCategoryCount, category)).collect(Collectors.toList());
		} else {
			return Collections.emptyList();
		}
		**/
		
		return null;
	}

	private ReadableCategory convertCategoryToReadableCategory(MerchantStore merchantStore, Language language,
			Map<String, Long> productCategoryCount, Category category) {
		ReadableCategoryPopulator populator = new ReadableCategoryPopulator();
		try {
			ReadableCategory categoryProxy = populator.populate(category, new ReadableCategory(), merchantStore,
					language);
			Long total = productCategoryCount.get(categoryProxy.getCode());
			if (total != null) {
				categoryProxy.setProductCount(total.intValue());
			}
			return categoryProxy;
		} catch (ConversionException e) {
			throw new ConversionRuntimeException(e);
		}
	}

	private ReadableProduct convertProductToReadableProduct(Product product, MerchantStore merchantStore,
			Language language) {

		ReadableProductPopulator populator = new ReadableProductPopulator();
		populator.setPricingService(pricingService);
		populator.setimageUtils(imageUtils);

		try {
			return populator.populate(product, new ReadableProduct(), merchantStore, language);
		} catch (ConversionException e) {
			throw new ConversionRuntimeException(e);
		}
	}

	@Override
	public ValueList autocompleteRequest(String word, MerchantStore store, Language language) {
		Validate.notNull(word,"Search Keyword must not be null");
		Validate.notNull(language, "Language cannot be null");
		Validate.notNull(store,"MerchantStore cannot be null");
		
		SearchRequest req = new SearchRequest();
		req.setLanguage(language.getCode());
		req.setStore(store.getCode().toLowerCase());
		req.setSearchString(word);
		req.setLanguage(language.getCode());
		
		SearchResponse response;
		try {
			response = searchService.searchKeywords(store, language.getCode(), req, AUTOCOMPLETE_ENTRIES_COUNT);
		} catch (ServiceException e) {
			throw new RuntimeException(e);
		}
	
		
		List<String> keywords = response.getItems().stream().map(i -> i.getSuggestions()).collect(Collectors.toList());
		
		ValueList valueList = new ValueList();
		valueList.setValues(keywords);
		
		return valueList;
		

	}


}



```
