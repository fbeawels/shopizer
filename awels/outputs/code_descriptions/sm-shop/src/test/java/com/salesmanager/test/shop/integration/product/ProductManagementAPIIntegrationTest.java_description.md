# ProductManagementAPIIntegrationTest.java

## Review

## 1. Summary

The **`ProductManagementAPIIntegrationTest`** class is a Spring‑Boot integration test suite for the product‑management REST layer of a shop application.  
Its primary goal is to exercise the public API for creating categories, products, options, option values, reviews and retrieving products. The tests are executed against a live Spring context (random port) and interact with the service via `RestTemplate` or the injected `TestRestTemplate`.

**Key components**

| Component | Purpose |
|-----------|---------|
| `PersistableCategory`, `PersistableProduct`, `PersistableProductPrice`, … | DTOs that represent the payload sent to the REST endpoints. |
| `TestRestTemplate` (inherited from `ServicesTestSupport`) | Convenience wrapper that already contains the base URL and authentication headers. |
| `RestTemplate` | Used in a few ignored tests where the URL is hard‑coded. |
| `HttpEntity` | Carries the payload plus HTTP headers (e.g. `Content-Type`, `Authorization`). |
| `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` | Boots a full application context on a random port so the integration tests hit the real HTTP stack. |
| `@RunWith(SpringRunner.class)` | JUnit 4 integration with Spring TestContext. |

The test class follows a simple “send request → assert response” pattern, but many of the test methods are currently ignored or use hard‑coded values.

---

## 2. Detailed Description

### Execution Flow

1. **Context initialisation** – Spring Boot starts the full application on a random port.  
2. **`testRestTemplate` injection** – inherited from `ServicesTestSupport`.  
3. **Each test** runs in isolation (JUnit4 guarantees a new instance per test).  
4. **Request preparation** – DTOs are populated with fixed values, wrapped in an `HttpEntity` with authentication headers.  
5. **HTTP call** – The request is sent to the appropriate endpoint (most use the auto‑configured `/api/v1/private/...`).  
6. **Response handling** – The body is mapped back to the same DTO type; assertions check HTTP status and non‑null IDs.  
7. **Cleanup** – Not required because each test uses a fresh random port and does not persist state beyond the request.

### Design Choices & Assumptions

| Decision | Rationale / Assumption |
|----------|------------------------|
| Using `@Ignore` on several tests | These are either not ready or rely on external data (e.g. a pre‑existing customer). |
| Hard‑coded URLs (`http://localhost:8080/…`) | These tests are legacy and do not use the injected `testRestTemplate`. |
| DTOs use default values (e.g., `ProductOptionType.Select`) | The tests aim to create the most common entities quickly. |
| No `@Before`/`@After` methods | Each test builds its own objects; the test suite does not rely on shared mutable state. |

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `createProductWithCategory()` | Creates a new category and a product linked to that category. | None | `void` | Sends two POST requests; asserts success. |
| `createProductReview()` | Creates a review for product 1 by customer 1. | None | `void` | POSTs review; asserts success. |
| `createOptionValue()` | Creates a new product option value. | None | `void` | Uses raw `RestTemplate` to POST; prints the new ID. |
| `createOption()` | Creates a new product option. | None | `void` | Similar to `createOptionValue`. |
| `getProducts()` | Retrieves products by category ID. | None | `void` | GETs product list; prints count. |
| `putProduct()` | Placeholder for update logic. | None | `void` | Empty. |
| `postProduct()` | Creates a product with detailed specs, prices, images, and descriptions. | None | `void` | Builds DTO, serialises to JSON, POSTs, prints result. |
| `extractBytes(File)` | Reads file bytes into a byte array. | `File imgPath` | `byte[]` | Reads file; throws `Exception` on I/O error. |

**Reusable utilities**

* `extractBytes(File)` – small helper for image upload tests.  
* The code repeatedly builds `ObjectWriter` and serialises DTOs; this could be extracted into a static helper method.

---

## 4. Dependencies

| Dependency | Type | Role |
|------------|------|------|
| `org.springframework.boot.test.context.SpringBootTest` | Spring Boot | Bootstraps full context for integration tests. |
| `org.springframework.test.context.junit4.SpringRunner` | Spring Test | Integrates JUnit4 with Spring. |
| `org.springframework.boot.test.web.client.TestRestTemplate` | Spring Boot | Simplifies HTTP calls with base URL and authentication. |
| `org.springframework.web.client.RestTemplate` | Spring Web | Manual HTTP client for legacy tests. |
| `com.fasterxml.jackson.databind.ObjectMapper/Writer` | Jackson | Serialises DTOs to JSON for debugging. |
| `org.hamcrest.*`, `org.junit.Assert.*` | JUnit & Hamcrest | Assertions. |
| `com.salesmanager.*` | Project‑specific | DTOs and constants for the shop application. |
| `org.junit.Ignore` | JUnit | Disables tests temporarily. |

All dependencies are third‑party except for standard Java (`java.io.*`, `java.math.*`, etc.). There are no platform‑specific assumptions beyond the need for a reachable HTTP server.

---

## 5. Additional Notes

### Strengths

* **Realistic integration testing** – Uses the same endpoints that the front‑end would call.
* **Clear DTO mapping** – The test data mirrors production models.
* **Extensible skeleton** – Many placeholders are present for future tests.

### Weaknesses & Edge Cases

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded URLs** in ignored tests | Breaks when the server port changes. | Use `testRestTemplate` or inject the base URL. |
| **Magic strings** (`"test-cat"`, `"PRODUCT12"`, `"en"`, etc.) | Hard to maintain and may conflict with existing data. | Parameterise or generate unique values (e.g., UUID). |
| **No cleanup** | Test data persists across runs if the database is not reset. | Use an in‑memory DB for tests or delete entities in `@After`. |
| **Ignored tests** | Critical functionality (review, option creation) is not exercised. | Remove `@Ignore` or provide required pre‑conditions. |
| **Manual JSON serialisation** for debugging | Duplicate logic, increases noise. | Keep serialisation in a dedicated helper or use `TestRestTemplate` that automatically serialises. |
| **Potential authentication failure** | Tests rely on `getHeader()`; if credentials change, all tests fail. | Externalise credentials to a test config file or use Spring Security test support. |
| **Hard‑coded image path** | Fails on other machines. | Use classpath resources or generate temporary files. |
| **Missing assertion on response body** | Tests only check status, not returned data. | Assert critical fields (e.g., `id`, `code`, `price`). |

### Future Enhancements

1. **Use `TestRestTemplate` everywhere** – eliminates hard‑coded URLs and handles headers centrally.  
2. **Parameterise test data** – use UUIDs or timestamps to avoid clashes.  
3. **Leverage Spring Security test utilities** – e.g., `@WithMockUser`, to simplify auth.  
4. **Introduce `@Transactional`** – rollback after each test to keep the DB clean.  
5. **Add negative tests** – invalid payloads, missing fields, etc., to validate error handling.  
6. **Replace JUnit 4 with JUnit 5** – modernise the test suite and benefit from `@BeforeEach`, `@AfterEach`.  
7. **Use `RestAssured` or Spring's `WebTestClient`** – more expressive assertions (e.g., JSON path).  
8. **Extract common helper methods** – building a product, category, or option into static factory methods to keep tests focused.  

---

### Summary

The test suite demonstrates a solid foundation for end‑to‑end validation of the product‑management REST layer. However, it currently relies on hard‑coded values and ignored tests, limiting its effectiveness. By refactoring to use the provided `TestRestTemplate`, parameterising data, and adding comprehensive assertions, the suite can become a reliable safety net for future changes.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.product;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.CREATED;

import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileInputStream;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import org.junit.Ignore;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.web.client.RestTemplate;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ObjectWriter;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionType;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.catalog.category.Category;
import com.salesmanager.shop.model.catalog.category.CategoryDescription;
import com.salesmanager.shop.model.catalog.category.PersistableCategory;
import com.salesmanager.shop.model.catalog.manufacturer.Manufacturer;
import com.salesmanager.shop.model.catalog.product.PersistableProductPrice;
import com.salesmanager.shop.model.catalog.product.PersistableProductReview;
import com.salesmanager.shop.model.catalog.product.ProductDescription;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.catalog.product.attribute.PersistableProductOption;
import com.salesmanager.shop.model.catalog.product.attribute.PersistableProductOptionValue;
import com.salesmanager.shop.model.catalog.product.attribute.ProductOptionDescription;
import com.salesmanager.shop.model.catalog.product.attribute.ProductOptionValueDescription;
import com.salesmanager.shop.model.catalog.product.product.PersistableProduct;
import com.salesmanager.shop.model.catalog.product.product.PersistableProductInventory;
import com.salesmanager.shop.model.catalog.product.product.ProductSpecification;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
public class ProductManagementAPIIntegrationTest extends ServicesTestSupport {

	private RestTemplate restTemplate;

	private Long testCategoryID;

	private Long testProductID;

	@Test
	public void createProductWithCategory() throws Exception {


		final PersistableCategory newCategory = new PersistableCategory();
		newCategory.setCode("test-cat");
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

		final PersistableProduct product = super.product("PRODUCT12");
		final ArrayList<Category> categories = new ArrayList<>();
		categories.add(cat);
		product.setCategories(categories);
		ProductSpecification specifications = new ProductSpecification();
		specifications.setManufacturer(
				com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer.DEFAULT_MANUFACTURER);
		product.setProductSpecifications(specifications);
		product.setPrice(BigDecimal.TEN);
		product.setSku("123ABC");
		final HttpEntity<PersistableProduct> entity = new HttpEntity<>(product, getHeader());

		final ResponseEntity<PersistableProduct> response = testRestTemplate.postForEntity(
				"/api/v1/private/product?store=" + Constants.DEFAULT_STORE, entity, PersistableProduct.class);
		assertThat(response.getStatusCode(), is(CREATED));
	}

	/**
	 * Creates a ProductReview requires an existing Customer and an existing
	 * Product
	 *
	 * @throws Exception
	 */
	@Ignore
	@Test
	public void createProductReview() throws Exception {

		final PersistableProductReview review = new PersistableProductReview();
		review.setCustomerId(1L);

		review.setProductId(1L);
		review.setLanguage("en");
		review.setRating(2D);// rating is on 5
		review.setDescription(
				"Not as good as expected. From what i understood that was supposed to be premium quality but unfortunately i had to return the item after one week... Verry disapointed !");
		review.setDate("2021-06-06");
		final HttpEntity<PersistableProductReview> entity = new HttpEntity<>(review, getHeader());

		final ResponseEntity<PersistableProductReview> response = testRestTemplate.postForEntity(
				"/api/v1/private/products/1/reviews?store=" + Constants.DEFAULT_STORE, entity,
				PersistableProductReview.class);

		final PersistableProductReview rev = response.getBody();
		assertThat(response.getStatusCode(), is(CREATED));
		assertNotNull(rev.getId());

	}

	/**
	 * Creates a product option value that can be used to create a product
	 * attribute when creating a new product
	 *
	 * @throws Exception
	 */
	@Test
	@Ignore
	public void createOptionValue() throws Exception {

		final ProductOptionValueDescription description = new ProductOptionValueDescription();
		description.setLanguage("en");
		description.setName("Red");

		final List<ProductOptionValueDescription> descriptions = new ArrayList<>();
		descriptions.add(description);

		final PersistableProductOptionValue optionValue = new PersistableProductOptionValue();
		optionValue.setOrder(1);
		optionValue.setCode("colorred");
		optionValue.setDescriptions(descriptions);

		final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
		final String json = writer.writeValueAsString(optionValue);

		System.out.println(json);

		/**
		 * { "descriptions" : [ { "name" : "Red", "description" : null,
		 * "friendlyUrl" : null, "keyWords" : null, "highlights" : null,
		 * "metaDescription" : null, "title" : null, "language" : "en", "id" : 0
		 * } ], "order" : 1, "code" : "color-red", "id" : 0 }
		 */

		restTemplate = new RestTemplate();

		final HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

		final ResponseEntity response = restTemplate.postForEntity(
				"http://localhost:8080/sm-shop/services/private/DEFAULT/product/optionValue", entity,
				PersistableProductOptionValue.class);

		final PersistableProductOptionValue opt = (PersistableProductOptionValue) response.getBody();
		System.out.println("New optionValue ID : " + opt.getId());

	}

	/**
	 * Creates a new ProductOption
	 *
	 * @throws Exception
	 */
	@Test
	@Ignore
	public void createOption() throws Exception {

		final ProductOptionDescription description = new ProductOptionDescription();
		description.setLanguage("en");
		description.setName("Color");

		final List<ProductOptionDescription> descriptions = new ArrayList<>();
		descriptions.add(description);

		final PersistableProductOption option = new PersistableProductOption();
		option.setOrder(1);
		option.setCode("color");
		option.setType(ProductOptionType.Select.name());
		option.setDescriptions(descriptions);

		final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
		final String json = writer.writeValueAsString(option);

		System.out.println(json);

		/**
		 * { "descriptions" : [ { "name" : "Color", "description" : null,
		 * "friendlyUrl" : null, "keyWords" : null, "highlights" : null,
		 * "metaDescription" : null, "title" : null, "language" : "en", "id" : 0
		 * } ], "type" : SELECT, "order" : 1, "code" : "color", "id" : 0 }
		 */

		restTemplate = new RestTemplate();

		final HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

		final ResponseEntity response = restTemplate.postForEntity(
				"http://localhost:8080/sm-shop/services/private/DEFAULT/product/option", entity,
				PersistableProductOption.class);

		final PersistableProductOption opt = (PersistableProductOption) response.getBody();
		System.out.println("New option ID : " + opt.getId());

	}

	@Test
	@Ignore
	public void getProducts() throws Exception {
		restTemplate = new RestTemplate();

		final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());

		final ResponseEntity<ReadableProduct[]> response = restTemplate.exchange(
				"http://localhost:8080/sm-shop/services/rest/products/DEFAULT/en/" + testCategoryID, HttpMethod.GET,
				httpEntity, ReadableProduct[].class);

		if (response.getStatusCode() != HttpStatus.OK) {
			throw new Exception();
		} else {
			System.out.println(response.getBody().length + " Product records found.");
		}
	}

	@Test
	@Ignore
	public void putProduct() throws Exception {
		restTemplate = new RestTemplate();

		// TODO: Put Product

	}

	@Test
	@Ignore
	public void postProduct() throws Exception {
		restTemplate = new RestTemplate();
		
		final String code = "abcdef";

		final PersistableProduct product = super.product(code);

		

		final String categoryCode = "ROOT";// root category

		final Category category = new Category();
		category.setCode(categoryCode);
		final List<Category> categories = new ArrayList<>();
		categories.add(category);

		final String manufacturer = "temple";
		final Manufacturer collection = new Manufacturer();
		collection.setCode(manufacturer);

		// core properties


		// product.setManufacturer(collection); //no manufacturer assigned for
		// now
		// product.setCategories(categories); //no category assigned for now

		product.setSortOrder(0);// set iterator as sort order
		product.setAvailable(true);// force availability
		product.setProductVirtual(false);// force tangible good
		product.setQuantityOrderMinimum(1);// force to 1 minimum when ordering
		product.setProductShipeable(true);// all items are shipeable

		/** images **/
		final String image = "/Users/carlsamson/Documents/csti/IMG_4626.jpg";
		// String image = "C:/personal/em/pictures-misc/IMG_2675.JPG";

		final File imgPath = new File(image);

		// PersistableImage persistableImage = new PersistableImage();

		// persistableImage.setBytes(this.extractBytes(imgPath));
		// persistableImage.setImageName(imgPath.getName());

		// List<PersistableImage> images = new ArrayList<PersistableImage>();
		// images.add(persistableImage);

		// product.setImages(images);

		ProductSpecification specifications = new ProductSpecification();
		specifications.setHeight(new BigDecimal(20));
		specifications.setLength(new BigDecimal(21));
		specifications.setWeight(new BigDecimal(22));
		specifications.setWidth(new BigDecimal(23));

		product.setProductSpecifications(specifications);
		
		
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

		final List<ProductDescription> descriptions = new ArrayList<>();

		// add english description
		ProductDescription description = new ProductDescription();
		description.setLanguage("en");
		description.setTitle("Statue Head");
		description.setName("Statue Head");
		description.setDescription("Statue Head");
		description.setFriendlyUrl("Statue-head");

		descriptions.add(description);

		// add french description
		description = new ProductDescription();
		description.setLanguage("fr");
		description.setTitle("Tête de Statue");
		description.setName("Tête de Statue");
		description.setDescription(description.getName());
		description.setFriendlyUrl("tete-de-Statue");
		//

		descriptions.add(description);

		product.setDescriptions(descriptions);

		final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
		final String json = writer.writeValueAsString(product);

		System.out.println(json);

		final HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

		// post to create category web service
		final ResponseEntity response = restTemplate.postForEntity("http://localhost:8080/api/v1/product", entity,
				PersistableProduct.class);

		final PersistableProduct prod = (PersistableProduct) response.getBody();

		System.out.println("---------------------");

	}


	/** private helper methods **/
	public byte[] extractBytes(final File imgPath) throws Exception {

		final FileInputStream fis = new FileInputStream(imgPath);

		final BufferedInputStream inputStream = new BufferedInputStream(fis);
		final byte[] fileBytes = new byte[(int) imgPath.length()];
		inputStream.read(fileBytes);
		inputStream.close();

		return fileBytes;

	}

}


```
