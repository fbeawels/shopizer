# SearchServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`SearchServiceImpl` is a Spring‐managed service that integrates with an external search engine (via `SearchModule`) to index, delete, and query `Product` objects for an online store. It builds a rich, language‑aware document that contains product details, inventory, attributes, variants, and related metadata (manufacturer, category, image, etc.) and sends it to the search index.

**Key components**

| Component | Role |
|-----------|------|
| `SearchModule` | Third‑party abstraction that talks to the search backend (e.g., Elasticsearch, Solr). |
| `SearchConfiguration` | Holds cluster information, credentials, language specific mappings & settings. |
| `ApplicationSearchConfiguration` | Spring `@ConfigurationProperties` bean that injects user‑defined search properties (hosts, cluster, languages). |
| `ProductInventoryService` | Provides inventory data for a product or variant. |
| `SearchService` (interface) | Declares the public API that the application uses. |

**Design patterns / frameworks**

* Spring Boot (`@Service`, `@EnableConfigurationProperties`, dependency injection).  
* Builder/Factory pattern is implicitly used by `SearchModule.configure(SearchConfiguration)`.  
* A simple *Adapter* pattern: the service translates domain objects (`Product`, `ProductDescription`, etc.) into the generic `IndexItem` expected by the search engine.  

---

## 2. Detailed Description  

### Flow of execution  

| Stage | What happens |
|-------|--------------|
| **Bean creation** | Spring instantiates `SearchServiceImpl`. Dependencies are injected automatically. |
| **`@PostConstruct init()`** | If a `SearchModule` bean is available and indexing is not disabled (`noIndex == false`), a `SearchConfiguration` is built via `config()` and the module is configured. |
| **Indexing a product (`index(...)`)** | 1. Checks that indexing is enabled and a module is present. 2. Loads all product languages. 3. Removes any existing documents for the product (by language & ID). 4. For each `ProductDescription` (i.e. each language) calls `indexProduct(...)`. 5. `indexProduct` constructs an `IndexItem`, pulls inventory, attributes, variants, manufacturer, category, image, reviews, and links, then hands it to `searchModule.index(...)`. |
| **Deletion (`deleteDocument(...)`)** | Simple wrapper that removes all language variants of a product from the index. |
| **Search (`search`, `searchKeywords`)** | Delegates to the module. Parameters such as language, entry count, and start index are not used directly in the call – they must be embedded in the `SearchRequest` passed by the caller. |
| **Fetching a single document (`getDocument`)** | Retrieves the `Document` for a product ID in a given language. |

### Assumptions & constraints

* The product’s `id` is mandatory for indexing.  
* The search backend supports language‑specific mapping and settings, provided as JSON files in the classpath (`search/MAPPINGS.json`, `search/SETTINGS*.json`).  
* Inventory is always available via `ProductInventoryService`.  
* The service is **not** thread‑safe for configuration mutation; however, configuration is built once at startup.  
* The module is expected to be optional – if missing, all methods quietly do nothing (or return `null`).  

### Architectural choices

* **Loose coupling**: The search logic is isolated behind `SearchModule`, making it easy to swap the underlying search engine.  
* **Per‑language handling**: Documents are indexed per language to allow full‑text search in multiple locales.  
* **Explicit mapping**: Rather than relying on auto‑detect, the service explicitly loads mapping files for each language, giving full control over field types.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side effects / Exceptions |
|--------|---------|------------|---------|---------------------------|
| `init()` | `@PostConstruct` – configures the search module at startup. | None | None | Logs error if configuration fails. |
| `index(MerchantStore, Product)` | Indexes a product (all descriptions & variants). | `store`, `product` | `void` | Throws `ServiceException` on failure. |
| `document(Long, List<String>, RequestOptions)` | Helper: retrieves existing documents for a product ID across languages. | `id`, `languages`, `options` | `List<Document>` | Swallows exceptions, prints stack trace (not ideal). |
| `indexProduct(...)` | Builds an `IndexItem` from a `ProductDescription` and related data, then indexes it. | `store`, `description`, `product`, `variants` | `void` | Throws `ServiceException`. |
| `config()` | Builds a `SearchConfiguration` (cluster, hosts, credentials, mappings & settings). | None | `SearchConfiguration` | Throws `Exception`. |
| `inventory(Product)` | Creates a map of inventory fields for a product. | `product` | `Map<String,String>` | Throws `Exception`. |
| `inventory(ProductVariant)` | Same as above but for a variant. | `product` | `Map<String,String>` | Throws `Exception`. |
| `variants(ProductVariant)` | Returns a map describing a variant (codes + optional VSKU). | `variant` | `Map<String,String>` | None. |
| `deleteDocument(MerchantStore, Product)` | Removes all language variants of a product from the index. | `store`, `product` | `void` | Throws `ServiceException`. |
| `searchKeywords(...)` | Executes a keyword search. | `store`, `language`, `search`, `entriesCount` | `SearchResponse` | Throws `ServiceException`. |
| `search(...)` | Executes a product search. | `store`, `language`, `search`, `entriesCount`, `startIndex` | `SearchResponse` | Throws `ServiceException`. |
| `getDocument(String, MerchantStore, Long)` | Retrieves a single document by product ID and language. | `language`, `store`, `productId` | `Optional<Document>` | Throws `ServiceException`. |
| `languages(Product)` | Collects language codes from a product’s descriptions. | `product` | `List<String>` | None. |
| `manufacturer(Manufacturer, String)` | Looks up a manufacturer’s name for a language. | `manufacturer`, `language` | `String` | None. |
| `category(Category, String)` | Looks up a category’s name for a language. | `category`, `language` | `String` | None. |
| `attributes(Product, String)` | Gathers all attribute name/value pairs for a product. | `product`, `language` | `Map<String,String>` | None. |
| `attribute(ProductAttribute, String)` | Builds a single attribute entry. | `attribute`, `language` | `Map<String,String>` | None. |
| `settings(SearchConfiguration, String)` | Loads language‑specific settings JSON into config. | `config`, `language` | None | Throws `Exception`. |
| `mappings(SearchConfiguration, String)` | Loads language‑specific mapping JSON into config. | `config`, `language` | None | Throws `Exception`. |
| `loadClassPathResource(String)` | Reads a file from the classpath and returns its content. | `file` | `String` | Throws `Exception`. |

---

## 4. Dependencies  

| External / Third‑Party | Role |
|------------------------|------|
| **Spring Framework** (`@Service`, `@Autowired`, `@PostConstruct`, `@Value`, `@EnableConfigurationProperties`) | Bean management & configuration. |
| **Spring Boot** (`@EnableConfigurationProperties`, `ClassPathResource`) | Simplifies property handling. |
| **Apache Commons** (`CollectionUtils`, `StringUtils`) | Utility helpers. |
| **Apache Commons Lang** (`Validate`) | Parameter validation. |
| **JSoup** (`Validate`) | Same as above (note: could use a different library). |
| **SLF4J** | Logging. |
| **`modules.commons.search`** | External search abstraction (provides `SearchModule`, `SearchConfiguration`, `SearchRequest`, etc.). |
| **Java NIO** (`Files`) | File reading. |
| **Java Streams** | Collection manipulation. |

All dependencies are standard Java SE / open‑source libraries; none are platform‑specific beyond the typical JDK 8+ runtime.

---

## 5. Additional Notes & Recommendations  

### 5.1. Error handling & null safety  

* `document()` swallows exceptions and prints a stack trace – this should be replaced with proper logging and rethrowing or graceful fallback.  
* `getDocument()` returns `null` instead of an empty `Optional`. This breaks the contract of returning an `Optional<Document>` and can lead to `NullPointerException`s downstream.  
* Many helper methods use `.get()` on streams (e.g., `manufacturer()`, `category()`, `attribute()`). If the collection is empty the code will throw `NoSuchElementException`. A safer approach is `findFirst().orElse(null)` or returning an optional.  

### 5.2. File loading strategy  

* `loadClassPathResource()` uses `ClassPathResource.getFile()`, which fails when the application is packaged as a JAR (resource inside the jar is not a file). Use `getInputStream()` and read via `IOUtils` instead.  

### 5.3. Thread safety & caching  

* Configuration is built once; however, the `settings` and `mappings` methods load files on every call. If these files are large or the service is highly concurrent, consider caching the loaded JSON strings.  

### 5.4. API design consistency  

* Methods that accept a `MerchantStore` but do not use it (`search`, `searchKeywords`, `getDocument`) should either make use of the store (e.g., to build tenant‑specific index names) or remove the parameter to avoid confusion.  
* `search()` and `searchKeywords()` ignore the `entriesCount` and `startIndex` parameters – if these are intended to be respected, they should be passed to the `SearchRequest` or handled inside the module.  

### 5.5. Logging & monitoring  

* Indexing and deletion operations could emit more granular logs (e.g., product ID, language, operation status) to aid debugging.  
* Consider adding metrics (e.g., number of documents indexed, time taken) via Micrometer or similar.  

### 5.6. Unit testability  

* The service is heavily dependent on external beans (`SearchModule`, `ProductInventoryService`). Use interfaces and inject mocks in tests.  
* Extract file‑loading logic into a dedicated helper that can be mocked.  

### 5.7. Future enhancements  

| Idea | Benefit |
|------|---------|
| **Bulk indexing** | Reduce round‑trips when many products change at once. |
| **Partial re‑indexing** | Only re‑index changed fields instead of whole document. |
| **Language fallback** | If a product description for a language is missing, fall back to default. |
| **Retry & back‑off** | For transient failures against the search backend. |
| **Better configuration** | Replace hardcoded mapping file names with `@ConfigurationProperties` so that new languages can be added without code changes. |
| **OpenTelemetry** | Trace indexing/search requests for observability. |

---

### 5.8. Summary of concerns  

| Concern | Impact | Suggested fix |
|---------|--------|---------------|
| `getDocument()` returns `null` | Potential `NullPointerException` | Return `Optional.empty()` or throw a custom exception. |
| Swallowing exceptions in `document()` | Hidden failures | Log the error and rethrow or wrap in `ServiceException`. |
| `loadClassPathResource()` uses `getFile()` | Breaks in JAR | Use `getInputStream()` and `IOUtils`. |
| Potential `NoSuchElementException` in `.get()` | Crashes on missing data | Use `orElse(null)` or `orElseThrow` with a clear message. |
| Unused `MerchantStore` in several methods | API confusion | Remove unused param or make use of it. |
| Search methods ignore pagination params | Wrong results | Pass `entriesCount` & `startIndex` into `SearchRequest`. |

Overall, the class fulfills its role as a bridge between domain objects and the search backend, but could benefit from tighter error handling, better resource management, and a few API clean‑ups to improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.search;

import java.io.File;
import java.nio.file.Files;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.stream.Collectors;

import javax.annotation.PostConstruct;
import javax.inject.Inject;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.jsoup.helper.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.configuration.ApplicationSearchConfiguration;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.inventory.ProductInventoryService;
import com.salesmanager.core.business.utils.CoreConfiguration;
import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.category.CategoryDescription;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionDescription;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionValueDescription;
import com.salesmanager.core.model.catalog.product.description.ProductDescription;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.catalog.product.inventory.ProductInventory;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.manufacturer.ManufacturerDescription;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.merchant.MerchantStore;

import modules.commons.search.SearchModule;
import modules.commons.search.configuration.SearchConfiguration;
import modules.commons.search.request.Document;
import modules.commons.search.request.IndexItem;
import modules.commons.search.request.RequestOptions;
import modules.commons.search.request.SearchRequest;
import modules.commons.search.request.SearchResponse;

@Service("productSearchService")
@EnableConfigurationProperties(value = ApplicationSearchConfiguration.class)
public class SearchServiceImpl implements com.salesmanager.core.business.services.search.SearchService {
	
	
    @Value("${search.noindex:false}")//skip indexing process
    private boolean noIndex;

	private static final Logger LOGGER = LoggerFactory.getLogger(SearchServiceImpl.class);

	private final static String INDEX_PRODUCTS = "INDEX_PRODUCTS";
	
	private final static String SETTINGS = "search/SETTINGS";
	
	private final static String PRODUCT_MAPPING_DEFAULT = "search/MAPPINGS.json";
	
	private final static String QTY = "QTY";
	private final static String PRICE = "PRICE";
	private final static String DISCOUNT_PRICE = "DISCOUNT";
	private final static String SKU = "SKU";
	private final static String VSKU = "VSKU";
	
	
	/**
	 * TODO properties file
	 */
	
	private final static String KEYWORDS_MAPPING_DEFAULT = "{\"properties\":"
			+ "      {\n\"id\": {\n"
			+ "        \"type\": \"long\"\n"
			+ "      }\n"
			+ "     }\n"
			+ "    }";	
	


	@Inject
	private CoreConfiguration configuration;

	@Autowired
	private ApplicationSearchConfiguration applicationSearchConfiguration;
	
	@Autowired
	private ProductInventoryService productInventoryService;

	@Autowired(required = false)
	private SearchModule searchModule;

	@PostConstruct
	public void init() throws Exception {

		/**
		 * Configure search module
		 */

		if (searchModule != null && !noIndex) {

			SearchConfiguration searchConfiguration = config();
			try {
				searchModule.configure(searchConfiguration);
			} catch (Exception e) {
				LOGGER.error("SearchModule cannot be configured [" + e.getMessage() + "]", e);
			}
		}
	}

	public void index(MerchantStore store, Product product) throws ServiceException {

		Validate.notNull(product.getId(), "Product.id cannot be null");

		if (configuration.getProperty(INDEX_PRODUCTS) == null
				|| configuration.getProperty(INDEX_PRODUCTS).equals(Constants.FALSE) || searchModule == null) {
			return;
		}

		List<String> languages = languages(product);

		// existing documents
		List<Document> documents;
		List<Map<String, String>> variants = null;
		try {
			documents = document(product.getId(), languages, RequestOptions.DO_NOT_FAIL_ON_NOT_FOUND);

				if (!CollectionUtils.isEmpty(product.getVariants())) {
					variants = new ArrayList<Map<String, String>>();
					variants = product.getVariants().stream().map(i -> variants(i)).collect(Collectors.toList());
				}
	
				if (!CollectionUtils.isEmpty(documents)) {
					if (documents.iterator().next() != null) {
						searchModule.delete(languages, product.getId());
					}
				}


		} catch (Exception e) {
			throw new ServiceException(e);
		}

		Set<ProductDescription> descriptions = product.getDescriptions();

		for (ProductDescription description : descriptions) {
			indexProduct(store, description, product, variants);
		}

	}

	private List<Document> document(Long id, List<String> languages, RequestOptions options) throws Exception {
		List<Optional<Document>> documents = null;
		try {
			documents = searchModule.getDocument(id, languages, options);
		} catch(Exception e) {
			e.printStackTrace();
		}
		
		for(Optional<Document> d : documents) {
			if(d == null) {//not allowed
				return Collections.emptyList();
			}
		}
		
		List<Document> filteredList = documents.stream().filter(Optional::isPresent).map(Optional::get)
				.collect(Collectors.toList());

		return filteredList;

	}

	private void indexProduct(MerchantStore store, ProductDescription description, Product product,
			List<Map<String, String>> variants) throws ServiceException {

		try {
			ProductImage image = null;
			if (!CollectionUtils.isEmpty(product.getImages())) {
				image = product.getImages().stream().filter(i -> i.isDefaultImage()).findFirst()
						.orElse(product.getImages().iterator().next());
			}
			
			/**
			 * Inventory
			 */
			
			/**
			 * SKU, QTY, PRICE, DISCOUNT
			 */
			
			List<Map<String, String>> itemInventory = new ArrayList<Map<String, String>>();
			
			itemInventory.add(inventory(product));
			
			if (!CollectionUtils.isEmpty(product.getVariants())) {
				for(ProductVariant variant : product.getVariants()) {
					itemInventory.add(inventory(variant));
				}
			}

			IndexItem item = new IndexItem();
			item.setId(product.getId());
			item.setStore(store.getCode().toLowerCase());
			item.setDescription(description.getDescription());
			item.setName(description.getName());
			item.setInventory(itemInventory);


			if (product.getManufacturer() != null) {
				item.setBrand(manufacturer(product.getManufacturer(), description.getLanguage().getCode()));
			}

			if (!CollectionUtils.isEmpty(product.getCategories())) {
				item.setCategory(
						category(product.getCategories().iterator().next(), description.getLanguage().getCode()));
			}

			if (!CollectionUtils.isEmpty(product.getAttributes())) {
				Map<String, String> attributes = attributes(product, description.getLanguage().getCode());
				item.setAttributes(attributes);
			}

			if (image != null) {
				item.setImage(image.getProductImage());
			}

			if (product.getProductReviewAvg() != null) {
				item.setReviews(product.getProductReviewAvg().toString());
			}

			if (!CollectionUtils.isEmpty(variants)) {
				item.setVariants(variants);
			}

			item.setLanguage(description.getLanguage().getCode());
			item.setLink(description.getSeUrl());

			searchModule.index(item);
		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	private SearchConfiguration config() throws Exception {

		SearchConfiguration config = new SearchConfiguration();
		config.setClusterName(applicationSearchConfiguration.getClusterName());
		config.setHosts(applicationSearchConfiguration.getHost());
		config.setCredentials(applicationSearchConfiguration.getCredentials());

		config.setLanguages(applicationSearchConfiguration.getSearchLanguages());
		
		config.getLanguages().stream().forEach(l -> {
			try {
				this.mappings(config,l);
			} catch (Exception e) {
				throw new IllegalStateException(e);
			}
		});
		config.getLanguages().stream().forEach(l -> {
			try {
				this.settings(config,l);
			} catch (Exception e) {
				throw new IllegalStateException(e);
			}
		});
		


		/**
		 * The mapping
		 */
		/*
		 * config.getProductMappings().put("variants", "nested");
		 * config.getProductMappings().put("attributes", "nested");
		 * config.getProductMappings().put("brand", "keyword");
		 * config.getProductMappings().put("store", "keyword");
		 * config.getProductMappings().put("reviews", "keyword");
		 * config.getProductMappings().put("image", "keyword");
		 * config.getProductMappings().put("category", "text");
		 * config.getProductMappings().put("name", "text");
		 * config.getProductMappings().put("description", "text");
		 * config.getProductMappings().put("price", "float");
		 * config.getProductMappings().put("id", "long");
		 
		config.getKeywordsMappings().put("store", "keyword");
		*/

		return config;

	}
	
	private Map<String, String> inventory(Product product) throws Exception {
		
		
		/**
		 * Default inventory
		 */
		
		ProductInventory inventory = productInventoryService.inventory(product);
		
		Map<String, String> inventoryMap = new HashMap<String, String>();
		inventoryMap.put(SKU, product.getSku());
		inventoryMap.put(QTY, String.valueOf(inventory.getQuantity()));
		inventoryMap.put(PRICE, String.valueOf(inventory.getPrice().getStringPrice()));
		if(inventory.getPrice().isDiscounted()) {
			inventoryMap.put(DISCOUNT_PRICE, String.valueOf(inventory.getPrice().getStringDiscountedPrice()));
		}
		
		
		return inventoryMap;
	}
	
	private Map<String, String> inventory(ProductVariant product) throws Exception {
		
		
		/**
		 * Default inventory
		 */
		
		ProductInventory inventory = productInventoryService.inventory(product);
		
		Map<String, String> inventoryMap = new HashMap<String, String>();
		inventoryMap.put(SKU, product.getSku());
		inventoryMap.put(QTY, String.valueOf(inventory.getQuantity()));
		inventoryMap.put(PRICE, String.valueOf(inventory.getPrice().getStringPrice()));
		if(inventory.getPrice().isDiscounted()) {
			inventoryMap.put(DISCOUNT_PRICE, String.valueOf(inventory.getPrice().getStringDiscountedPrice()));
		}
		
		
		return inventoryMap;
	}

	private Map<String, String> variants(ProductVariant variant) {
		if (variant == null)
			return null;

		Map<String, String> variantMap = new HashMap<String, String>();
		if (variant.getVariation() != null) {
			String variantCode = variant.getVariation().getProductOption().getCode();
			String variantValueCode = variant.getVariation().getProductOptionValue().getCode();

			variantMap.put(variantCode, variantValueCode);

		}
		

		if (variant.getVariationValue() != null) {
			String variantCode = variant.getVariationValue().getProductOption().getCode();
			String variantValueCode = variant.getVariationValue().getProductOptionValue().getCode();
			variantMap.put(variantCode, variantValueCode);
		}
		
		if(!StringUtils.isBlank(variant.getSku())) {
			variantMap.put(VSKU, variant.getSku());
		}
		


		return variantMap;

	}

	@Override
	public void deleteDocument(MerchantStore store, Product product) throws ServiceException {

		if (configuration.getProperty(INDEX_PRODUCTS) == null
				|| configuration.getProperty(INDEX_PRODUCTS).equals(Constants.FALSE) || searchModule == null) {
			return;
		}

		List<String> languages = languages(product);

		try {
			searchModule.delete(languages, product.getId());
		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	@Override
	public SearchResponse searchKeywords(MerchantStore store, String language, SearchRequest search, int entriesCount)
			throws ServiceException {

		if (configuration.getProperty(INDEX_PRODUCTS) == null
				|| configuration.getProperty(INDEX_PRODUCTS).equals(Constants.FALSE) || searchModule == null) {
			return null;
		}

		try {
			return searchModule.searchKeywords(search);
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	}

	@Override
	public SearchResponse search(MerchantStore store, String language, SearchRequest search, int entriesCount,
			int startIndex) throws ServiceException {

		if (configuration.getProperty(INDEX_PRODUCTS) == null
				|| configuration.getProperty(INDEX_PRODUCTS).equals(Constants.FALSE) || searchModule == null) {
			return null;
		}

		try {
			return searchModule.searchProducts(search);
		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	@Override
	public Optional<Document> getDocument(String language, MerchantStore store, Long productId)
			throws ServiceException {
		if (configuration.getProperty(INDEX_PRODUCTS) == null
				|| configuration.getProperty(INDEX_PRODUCTS).equals(Constants.FALSE) || searchModule == null) {
			return null;
		}

		try {
			return searchModule.getDocument(productId, language, RequestOptions.DEFAULT);
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	}

	private List<String> languages(Product product) {
		return product.getDescriptions().stream().map(d -> d.getLanguage().getCode()).collect(Collectors.toList());
	}

	private String manufacturer(Manufacturer manufacturer, String language) {
		ManufacturerDescription description = manufacturer.getDescriptions().stream()
				.filter(d -> d.getLanguage().getCode().equals(language)).findFirst().get();
		return description.getName();
	}

	private String category(Category category, String language) {
		CategoryDescription description = category.getDescriptions().stream()
				.filter(d -> d.getLanguage().getCode().equals(language)).findFirst().get();
		return description.getName();
	}

	private Map<String, String> attributes(Product product, String language) {

		/**
		 * ProductAttribute ProductOption ProductOptionValue
		 */

		Map<String, String> allAttributes = new HashMap<String, String>();

		for (ProductAttribute attribute : product.getAttributes()) {
			Map<String, String> attr = attribute(attribute, language);
			allAttributes.putAll(attr);
		}

		return allAttributes;

	}

	private Map<String, String> attribute(ProductAttribute attribute, String language) {
		Map<String, String> attributeValue = new HashMap<String, String>();

		ProductOptionDescription optionDescription = attribute.getProductOption().getDescriptions().stream()
				.filter(a -> a.getLanguage().getCode().equals(language)).findFirst().get();		
				
		ProductOptionValueDescription value = attribute.getProductOptionValue().getDescriptions().stream()
				.filter(a -> a.getLanguage().getCode().equals(language)).findFirst().get();

		attributeValue.put(optionDescription.getName(), value.getName());

		return attributeValue;
	}
	
	private void settings(SearchConfiguration config, String language) throws Exception{
		Validate.notEmpty(language, "Configuration requires language");
		String settings = loadClassPathResource(SETTINGS + "_DEFAULT.json");
		//specific settings
		if(language.equals("en")) {
			settings = loadClassPathResource(SETTINGS+ "_" + language +".json");
		}
		
		config.getSettings().put(language, settings);

	}
	
	private void mappings(SearchConfiguration config, String language) throws Exception {
		Validate.notEmpty(language, "Configuration requires language");


		config.getProductMappings().put(language, loadClassPathResource(PRODUCT_MAPPING_DEFAULT));
		config.getKeywordsMappings().put(language, KEYWORDS_MAPPING_DEFAULT);
			
	}
	
	public String loadClassPathResource(String file) throws Exception {
		Resource res = new ClassPathResource(file);
		File f = res.getFile();
		
		return new String(
			      Files.readAllBytes(f.toPath()));
	}

}



```
