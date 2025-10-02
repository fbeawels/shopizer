# ServicesTestSupport.java

## Review

## 1. Summary  

**Purpose** – `ServicesTestSupport` is a Spring‑Boot integration‑test helper that centralises common setup and HTTP‑client logic for a Shopizer‑based e‑commerce backend.  It abstracts repetitive tasks such as logging in, constructing authentication headers, and building demo domain objects (manufacturers, categories, products, shopping carts) so that test classes can focus on business‑logic verification rather than boilerplate.

**Key Components**  
| Component | Role |
|-----------|------|
| `TestRestTemplate` | Performs HTTP requests against the embedded test server (`WebEnvironment.RANDOM_PORT`). |
| `getHeader(...)` | Logs in with a supplied or default user and returns a populated `HttpHeaders` instance (Bearer token). |
| `fetchStore()` / `fetchCustomers()` | Convenience wrappers for reading store and customer data via REST. |
| `manufacturer(...)`, `category(...)` | Build *persistable* domain objects that can be sent to the API to create entities. |
| `product(...)` | Builds a *persistable* product with inventory, pricing, and a basic description. |
| `sampleProduct(...)` | End‑to‑end helper that creates a category, creates a product inside that category, then retrieves the product via the public API. |
| `sampleCart()` | Creates a shopping‑cart item for a newly created product and returns the cart state. |

**Design Patterns & Frameworks**  
* **Spring Boot Test** – The class is annotated with `@SpringBootTest` and `@ExtendWith(SpringExtension.class)` to leverage Spring’s test infrastructure.  
* **Fluent Builder‑style** – The helper methods return fully‑initialised domain objects, following a simple builder pattern.  
* **Test Rest Template** – `TestRestTemplate` is used instead of raw `RestTemplate` to automatically handle the random port and base URL resolution.

No external libraries beyond Spring Boot, JUnit 5, and Hamcrest are used.

---

## 2. Detailed Description  

### Execution Flow  
1. **Test Initialization** – Spring boots the `ShopApplication` on a random port, wiring `TestRestTemplate`.  
2. **Authentication** – Every helper that needs to hit a protected endpoint calls `getHeader()`, which performs a `/api/v1/private/login` POST with a username/password and extracts the JWT token from the response.  
3. **Domain Creation** –  
   * `sampleProduct(...)` first builds a `PersistableCategory` (via `category(...)`), POSTs it to `/api/v1/private/category?store=…`, then builds a `PersistableProduct` (via `product(...)`), attaches the created category, and POSTs it to `/api/v1/private/product?store=…`.  
   * `sampleCart()` reuses `sampleProduct(...)` to obtain a product, builds a `PersistableShoppingCartItem`, and POSTs it to `/api/v1/cart/`.  
4. **Assertions** – After each POST, the code checks the HTTP status (CREATED) and verifies that the returned entity contains an ID.  
5. **Retrieval** – For `sampleProduct(...)` the code performs a GET against `/api/v2/product/{code}` and validates a 200 OK.  

### Assumptions & Constraints  
* **Hard‑coded credentials** – `admin@shopizer.com / password` are used by default; tests cannot easily run under different user contexts without overriding `getHeader(user, pass)`.  
* **Single store** – All API calls assume the default store (`Constants.DEFAULT_STORE`).  
* **No cleanup** – Created categories/products/carts are never deleted, leading to state leakage between tests.  
* **No timeout/exception handling** – All REST calls assume success; any failure will throw an exception that propagates to the test framework.  

### Architecture  
The class is deliberately simple: it is *not* a service layer component but a **test support bean**.  It couples tightly to the API contract of the Shopizer backend, exposing a minimal domain‑specific façade that tests can use.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `getHeader()` | Log in with default admin credentials and return HTTP headers with a Bearer token. | – | `HttpHeaders` | Performs a POST to `/api/v1/private/login`. |
| `getHeader(String userName, String password)` | Same as above but with custom credentials. | `userName`, `password` | `HttpHeaders` | Performs a POST to `/api/v1/private/login`. |
| `fetchStore()` | Retrieve the default store details. | – | `ReadableMerchantStore` | GET `/api/v1/store/{storeCode}`. |
| `fetchCustomers()` | Retrieve the list of customers. | – | `ReadableCustomerList` | GET `/api/v1/private/customers`. |
| `manufacturer(String code)` | Build a `PersistableManufacturer` with a single description. | `code` | `PersistableManufacturer` | None (purely object construction). |
| `category(String code)` | Build a `PersistableCategory` with a single English description. | `code` | `PersistableCategory` | None. |
| `category(String code, String name)` | Build a `PersistableCategory` with a description that includes a friendly URL. | `code`, `name` | `PersistableCategory` | None. |
| `product(String code)` | Build a `PersistableProduct` with inventory, price, and a description. | `code` | `PersistableProduct` | None. |
| `sampleProduct(String code)` | **Full flow** – create a category, create a product within that category, retrieve the public representation. | `code` | `ReadableProduct` | Creates resources via the API and deletes nothing. |
| `sampleCart()` | Create a shopping‑cart item for a freshly created product and return the cart state. | – | `ReadableShoppingCart` | Creates a cart item via the API. |

**Reusable / Utility**  
`getHeader(...)` is the core utility method reused across all helpers.  The various `*` methods that build domain objects (`manufacturer`, `category`, `product`) can also be used directly by tests that need to POST their own entities.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.boot:spring-boot-starter-test` | Third‑party | Provides JUnit 5, Hamcrest, and `TestRestTemplate`. |
| `org.springframework.boot:spring-boot-starter-web` | Third‑party | Needed for `RestTemplate`/`TestRestTemplate`. |
| `org.junit.jupiter:junit-jupiter-api` | Third‑party | Core JUnit 5. |
| `org.hamcrest:hamcrest-core` | Third‑party | Assertion utilities. |
| `com.salesmanager.core` / `com.salesmanager.shop` | Application | Domain classes (Category, Product, etc.). |
| `org.springframework.test.context.junit.jupiter.SpringExtension` | Standard | Spring test extension. |

All dependencies are standard to a Spring Boot application; there are no OS‑specific or external service dependencies.  The only real runtime assumption is that the application can be started and that the authentication endpoint responds.

---

## 5. Additional Notes  

### Strengths  
* **Centralised authentication** – One call to `getHeader()` keeps token management consistent across tests.  
* **Readable API** – Helper methods hide HTTP details, making test code more declarative.  
* **Ease of reuse** – Tests can simply call `sampleProduct()` or `sampleCart()` to obtain a ready‑to‑use entity.  

### Weaknesses / Edge Cases  
1. **State leakage** – No teardown logic means repeated test runs create duplicate categories/products/carts, potentially exhausting DB or causing failures due to duplicate keys.  
2. **Hard‑coded store & language** – The helpers assume a single store (`Constants.DEFAULT_STORE`) and English language.  Tests that need other locales or multi‑store contexts will need to modify this class.  
3. **No error handling** – If login fails, `response.getBody()` will be `null`, leading to `NullPointerException` when trying to access the token.  Similarly, any non‑2xx response will propagate as an exception.  
4. **Token caching** – Each call to `getHeader()` performs a fresh login, which is unnecessary and may overload the auth service.  Caching a token until it expires would be more efficient.  
5. **Thread safety** – The class is a Spring bean, but the helper methods are not thread‑safe if used concurrently (e.g., shared mutable state in `Persistable*` objects).  Typically tests run sequentially, but parallel execution could expose race conditions.  

### Suggested Enhancements  
| Area | Recommendation |
|------|----------------|
| **Token Management** | Cache the JWT token per test class or per `TestRestTemplate` instance and refresh only when needed. |
| **Cleanup** | Implement a `@AfterEach` method that deletes created categories/products/carts or run tests in an isolated in‑memory database that resets per test. |
| **Parameterisation** | Add overloads or builder‑style methods that accept store code, language, and other metadata, enabling more flexible test scenarios. |
| **Error Reporting** | Wrap REST calls in helper methods that check status codes and throw informative exceptions (including response body). |
| **Logging** | Log authentication and creation steps to aid debugging of flaky tests. |
| **Parallelism** | Mark methods as `synchronized` or refactor to use local objects so that parallel test execution is safe. |

Overall, `ServicesTestSupport` is a solid foundation for integration testing against the Shopizer API, but it would benefit from improved state management, error handling, and configurability to support a broader range of test scenarios.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.common;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.CREATED;
import static org.springframework.http.HttpStatus.OK;

import java.math.BigDecimal;
import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.catalog.category.Category;
import com.salesmanager.shop.model.catalog.category.CategoryDescription;
import com.salesmanager.shop.model.catalog.category.PersistableCategory;
import com.salesmanager.shop.model.catalog.manufacturer.ManufacturerDescription;
import com.salesmanager.shop.model.catalog.manufacturer.PersistableManufacturer;
import com.salesmanager.shop.model.catalog.product.PersistableProductPrice;
import com.salesmanager.shop.model.catalog.product.ProductDescription;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.catalog.product.product.PersistableProduct;
import com.salesmanager.shop.model.catalog.product.product.PersistableProductInventory;
import com.salesmanager.shop.model.catalog.product.product.ProductSpecification;
import com.salesmanager.shop.model.entity.Entity;
import com.salesmanager.shop.model.shoppingcart.PersistableShoppingCartItem;
import com.salesmanager.shop.model.shoppingcart.ReadableShoppingCart;
import com.salesmanager.shop.model.store.ReadableMerchantStore;
import com.salesmanager.shop.populator.customer.ReadableCustomerList;
import com.salesmanager.shop.store.security.AuthenticationRequest;
import com.salesmanager.shop.store.security.AuthenticationResponse;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@ExtendWith(SpringExtension.class)
public class ServicesTestSupport {

	@Autowired
	protected TestRestTemplate testRestTemplate;

	protected HttpHeaders getHeader() {
		return getHeader("admin@shopizer.com", "password");
	}

	protected HttpHeaders getHeader(final String userName, final String password) {
		final ResponseEntity<AuthenticationResponse> response = testRestTemplate.postForEntity("/api/v1/private/login",
				new HttpEntity<>(new AuthenticationRequest(userName, password)), AuthenticationResponse.class);
		final HttpHeaders headers = new HttpHeaders();
		headers.setContentType(new MediaType("application", "json", Charset.forName("UTF-8")));
		headers.add("Authorization", "Bearer " + response.getBody().getToken());
		return headers;
	}

	public ReadableMerchantStore fetchStore() {
		final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());
		return testRestTemplate.exchange(String.format("/api/v1/store/%s", Constants.DEFAULT_STORE), HttpMethod.GET,
				httpEntity, ReadableMerchantStore.class).getBody();

	}

	public ReadableCustomerList fetchCustomers() {
		final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());
		return testRestTemplate
				.exchange("/api/v1/private/customers", HttpMethod.GET, httpEntity, ReadableCustomerList.class)
				.getBody();

	}

	protected PersistableManufacturer manufacturer(String code) {

		PersistableManufacturer m = new PersistableManufacturer();
		m.setCode(code);
		m.setOrder(0);

		ManufacturerDescription desc = new ManufacturerDescription();
		desc.setLanguage("en");
		desc.setName(code);

		m.getDescriptions().add(desc);

		return m;

	}

	protected PersistableCategory category(String code) {

		PersistableCategory newCategory = new PersistableCategory();
		newCategory.setCode(code);
		newCategory.setSortOrder(1);
		newCategory.setVisible(true);
		newCategory.setDepth(1);

		CategoryDescription description = new CategoryDescription();
		description.setLanguage("en");
		description.setName(code);

		List<CategoryDescription> descriptions = new ArrayList<>();
		descriptions.add(description);

		newCategory.setDescriptions(descriptions);

		return newCategory;

	}
	
	protected PersistableCategory category(String code, String name) {

		PersistableCategory newCategory = category(code);


		CategoryDescription description = new CategoryDescription();
		description.setLanguage("en");
		description.setName(name);
		description.setFriendlyUrl(name);

		List<CategoryDescription> descriptions = new ArrayList<>();
		descriptions.add(description);

		newCategory.setDescriptions(descriptions);

		return newCategory;

	}

	protected PersistableProduct product(String code) {

		PersistableProduct product = new PersistableProduct();

		PersistableProductInventory inventory = new PersistableProductInventory();
		inventory.setQuantity(5);
		inventory.setSku(code);

		final PersistableProductPrice productPrice = new PersistableProductPrice();
		productPrice.setDefaultPrice(true);

		productPrice.setPrice(new BigDecimal(250));
		productPrice.setDiscountedPrice(new BigDecimal(125));

		final List<PersistableProductPrice> productPriceList = new ArrayList<>();
		productPriceList.add(productPrice);

		inventory.setPrice(productPrice);
		product.setInventory(inventory);


		ProductDescription description = new ProductDescription();
		description.setName(code);
		description.setLanguage("en");

		product.getDescriptions().add(description);

		return product;

	}

	protected ReadableProduct sampleProduct(String code) {

		final PersistableCategory newCategory = new PersistableCategory();
		newCategory.setCode(code);
		newCategory.setSortOrder(1);
		newCategory.setVisible(true);
		newCategory.setDepth(4);

		final Category parent = new Category();

		newCategory.setParent(parent);

		final CategoryDescription description = new CategoryDescription();
		description.setLanguage("en");
		description.setName("test-cat");
		description.setFriendlyUrl("test-cat");
		description.setTitle("test-cat");

		final List<CategoryDescription> descriptions = new ArrayList<>();
		descriptions.add(description);

		newCategory.setDescriptions(descriptions);

		final HttpEntity<PersistableCategory> categoryEntity = new HttpEntity<>(newCategory, getHeader());

		final ResponseEntity<PersistableCategory> categoryResponse = testRestTemplate.postForEntity(
				"/api/v1/private/category?store=" + Constants.DEFAULT_STORE, categoryEntity, PersistableCategory.class);
		final PersistableCategory cat = categoryResponse.getBody();
		assertThat(categoryResponse.getStatusCode(), is(CREATED));
		assertNotNull(cat.getId());

		final PersistableProduct product = this.product(code);
		final ArrayList<Category> categories = new ArrayList<>();
		categories.add(cat);
		product.setCategories(categories);
		ProductSpecification specifications = new ProductSpecification();
		specifications.setManufacturer(
				com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer.DEFAULT_MANUFACTURER);
		product.setProductSpecifications(specifications);
		product.setAvailable(true);
		product.setPrice(BigDecimal.TEN);
		product.setSku(code);
		product.setQuantity(100);
		/**
		ProductDescription productDescription = new ProductDescription();
		productDescription.setDescription("TEST");
		productDescription.setName("TestName");
		productDescription.setLanguage("en");
		product.getDescriptions().add(productDescription);
		**/

		final HttpEntity<PersistableProduct> entity = new HttpEntity<>(product, getHeader());

		final ResponseEntity<Entity> response = testRestTemplate.postForEntity(
				"/api/v1/private/product?store=" + Constants.DEFAULT_STORE, entity, Entity.class);
		assertThat(response.getStatusCode(), is(CREATED));

		final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());

		String apiUrl = "/api/v2/product/" + code;

		ResponseEntity<ReadableProduct> readableProduct = testRestTemplate.exchange(apiUrl, HttpMethod.GET, httpEntity,
				ReadableProduct.class);
		assertThat(readableProduct.getStatusCode(), is(OK));

		return readableProduct.getBody();
	}

	protected ReadableShoppingCart sampleCart() {

		ReadableProduct product = sampleProduct("sampleCart");
		assertNotNull(product);

		PersistableShoppingCartItem cartItem = new PersistableShoppingCartItem();
		cartItem.setProduct(product.getSku());
		cartItem.setQuantity(1);

		final HttpEntity<PersistableShoppingCartItem> cartEntity = new HttpEntity<>(cartItem, getHeader());
		final ResponseEntity<ReadableShoppingCart> response = testRestTemplate
				.postForEntity(String.format("/api/v1/cart/"), cartEntity, ReadableShoppingCart.class);

		assertNotNull(response);
		assertThat(response.getStatusCode(), is(CREATED));

		return response.getBody();
	}

}



```
