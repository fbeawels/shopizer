# CategoryManagementAPIIntegrationTest.java

## Review

## 1. Summary

This class, **`CategoryManagementAPIIntegrationTest`**, is a Spring‑Boot integration test suite for the category REST API of a shop application.  
The tests exercise the main CRUD operations on categories, the creation of complex nested categories, and the relationship between categories, manufacturers and products.  
Key components:

| Component | Role |
|-----------|------|
| `TestRestTemplate` | Drives HTTP requests against the randomly‑started test server |
| `Persistable*` / `Readable*` models | DTOs used for request/response payloads |
| `ServicesTestSupport` | Provides helper methods (`getHeader()`, `manufacturer()`, `product()`, `category()`) and common constants |

The test suite relies on Spring’s `@SpringBootTest` infrastructure and JUnit 4 (`@RunWith(SpringRunner.class)`).

---

## 2. Detailed Description

### Execution flow

1. **Spring Boot context** is bootstrapped with `ShopApplication.class` on a random port.  
2. **`TestRestTemplate`** is injected; it automatically knows the port and base URL.  
3. Each test method sends HTTP requests to the REST API endpoints, deserialises the JSON payloads into the appropriate DTOs, and makes assertions about status codes, response bodies, and side effects.  

### Core interactions

| Test | Endpoint | HTTP Verb | Purpose |
|------|----------|-----------|---------|
| `getCategory` | `/api/v1/category/` | GET | Retrieve list of categories |
| `postCategory` | `/api/v1/private/category` | POST | Create a single category |
| `postComplexCategory` | `/api/v1/private/category` | POST | Create a nested category tree |
| `deleteCategory` | `/services/DEFAULT/category/100` | DELETE | Delete a category by ID |
| `manufacturerForItemsInCategory` | `/api/v1/private/manufacturer`, `/api/v1/private/category`, `/api/v1/private/product`, `/api/v1/category/{id}/manufacturer` | POST/GET | Verify that manufacturers referenced by products in a category are returned |

### Design choices & assumptions

* **Use of raw types** – Several `ResponseEntity` declarations omit generics (`ResponseEntity response`) which forces casts and suppresses warnings.  
* **Hard‑coded IDs** – `deleteCategory` uses `100` as a hard‑coded identifier; this assumes such a record already exists in the test database.  
* **No cleanup** – Tests create data (manufacturers, categories, products) but never delete it. While Spring’s test context is reset for each test, the underlying database (unless in-memory) may accumulate records across test runs.  
* **Commented code** – Several tests (`putCategory`, `getByCategoryFriendlyUrl`) are commented out, hinting at incomplete or deprecated functionality.  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `getCategory()` | Calls GET `/api/v1/category/` and verifies a non‑null list of `ReadableCategory` objects. | None | None | None |
| `postCategory()` | Builds a `PersistableCategory`, POSTs it to `/api/v1/private/category`, asserts `201 CREATED` and that an ID is returned. | None | None | Persists a category |
| `postComplexCategory()` | Constructs a multi‑level category tree, POSTs it, and verifies creation. | None | None | Persists a complex category hierarchy |
| `deleteCategory()` | Sends DELETE to `/services/DEFAULT/category/100`. No assertions. | None | None | Deletes the specified category (if it exists) |
| `manufacturerForItemsInCategory()` | Creates two manufacturers, a category, two products each linked to a different manufacturer, then requests the list of manufacturers associated with the category. | None | None | Persists manufacturers, category, products; fetches list of manufacturers |
| *Commented out methods* | `putCategory()`, `getByCategoryFriendlyUrl()` | None | None | Update and lookup by friendly URL (currently inactive) |

All test methods are annotated with `@Test` (except the commented ones) and declare `throws Exception` to propagate any failure.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.boot:spring-boot-starter-test` | Test framework | Provides JUnit 4, Spring Test, TestRestTemplate, Hamcrest |
| `org.springframework.boot:spring-boot-test` | Core | Starts the application context |
| `com.fasterxml.jackson.core:jackson-databind` | JSON | Serialises/deserialises DTOs |
| `com.salesmanager.core:...` | Internal | `Constants` class, possibly other core utilities |
| `com.salesmanager.shop:...` | Internal | Models (`PersistableCategory`, `ReadableCategory`, etc.) and `ShopApplication` |
| `org.junit:junit` | Testing | JUnit 4 |
| `org.hamcrest:hamcrest-core` | Assertions | `assertThat`, `is` |

All dependencies are either part of Spring Boot’s starter or internal to the SalesManager codebase.

---

## 5. Additional Notes & Recommendations

### Strengths
* **End‑to‑end coverage** – The tests exercise the API through HTTP rather than service layers, providing confidence that the controller, DTO mapping, and persistence layers work together.
* **Use of DTOs** – Clear separation between the request/response models (`Persistable*`, `Readable*`) keeps the tests readable.

### Weaknesses & Edge Cases
1. **Lack of assertions on DELETE** – `deleteCategory()` sends a DELETE but never verifies the outcome (status code, empty body, or subsequent GET failure).  
2. **Hard‑coded IDs & paths** – Using `/services/DEFAULT/category/100` and hard‑coded friendly URLs can make tests fragile if the underlying data changes.  
3. **Raw types & suppressed warnings** – Several `ResponseEntity` instances are declared without generics and use `@SuppressWarnings("rawtypes")`. This hides potential type‑safety issues.  
4. **No cleanup / isolation** – Tests rely on the shared test database; repeated runs may leave stale data or fail if the state changes. Consider using an in‑memory database (`H2`) or annotating tests with `@Transactional`/`@DirtiesContext` to reset state.  
5. **Commented‑out tests** – The presence of large blocks of commented code indicates either incomplete features or a need for refactoring. They should be removed or properly enabled.  
6. **Missing validation of response body structure** – For example, `postCategory()` only checks that `id` is not null. It would be more robust to validate the full payload (code, friendlyUrl, etc.).  
7. **Endpoint naming inconsistency** – The test uses both `/api/v1/category` and `/services/DEFAULT/category`. The API contract should be uniform.  
8. **Error handling** – `getCategory()` throws an exception if status ≠ OK; this is fine but could be replaced with JUnit assertions for readability.  

### Future Enhancements
- **Parameterise test data** – Use JUnit `@ParameterizedTest` or a data provider to run the same logic for multiple category codes/ names.  
- **Schema validation** – Integrate JSON schema validation for responses to catch structural regressions.  
- **Cleanup logic** – After creating manufacturers, categories, and products, delete them or rely on transactional rollback.  
- **Better assertion helpers** – Create utility methods like `assertResponseOk(ResponseEntity<?>)` to reduce boilerplate.  
- **Enable and refactor commented tests** – Re‑enable `putCategory()` and `getByCategoryFriendlyUrl()` once the underlying API supports those operations.  
- **Use of `MockMvc`** – For pure controller‑layer tests, `MockMvc` can speed up execution compared to full HTTP calls.  

---

**Overall Verdict** – The test suite demonstrates solid integration testing practice but can benefit from tighter type safety, more comprehensive assertions, and better test isolation. Addressing the points above will make the tests more reliable, maintainable, and expressive.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.category;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertTrue;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.CREATED;
import static org.springframework.http.HttpStatus.OK;
import java.util.ArrayList;
import java.util.List;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ObjectWriter;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.catalog.category.Category;
import com.salesmanager.shop.model.catalog.category.CategoryDescription;
import com.salesmanager.shop.model.catalog.category.PersistableCategory;
import com.salesmanager.shop.model.catalog.category.ReadableCategory;
import com.salesmanager.shop.model.catalog.category.ReadableCategoryList;
import com.salesmanager.shop.model.catalog.manufacturer.PersistableManufacturer;
import com.salesmanager.shop.model.catalog.manufacturer.ReadableManufacturer;
import com.salesmanager.shop.model.catalog.product.product.PersistableProduct;
import com.salesmanager.shop.model.catalog.product.product.ProductSpecification;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
public class CategoryManagementAPIIntegrationTest extends ServicesTestSupport {

    @Autowired
    private TestRestTemplate testRestTemplate;



    /**
     * Read - GET a category by id
     *
     * @throws Exception
     */
    @Test
    public void getCategory() throws Exception {
        final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());

        final ResponseEntity<ReadableCategoryList> response = testRestTemplate.exchange(String.format("/api/v1/category/"), HttpMethod.GET,
                httpEntity, ReadableCategoryList.class);
        if (response.getStatusCode() != HttpStatus.OK) {
            throw new Exception(response.toString());
        } else {
            final List<ReadableCategory> categories = response.getBody().getCategories();
            assertNotNull(categories);
        }
    }

    /**
     * Creates - POST a category for a given store
     *
     * @throws Exception
     */

    @Test
    public void postCategory() throws Exception {

        PersistableCategory newCategory = new PersistableCategory();
        newCategory.setCode("javascript");
        newCategory.setSortOrder(1);
        newCategory.setVisible(true);
        newCategory.setDepth(4);

        Category parent = new Category();

        newCategory.setParent(parent);

        CategoryDescription description = new CategoryDescription();
        description.setLanguage("en");
        description.setName("Javascript");
        description.setFriendlyUrl("javascript");
        description.setTitle("Javascript");

        List<CategoryDescription> descriptions = new ArrayList<>();
        descriptions.add(description);

        newCategory.setDescriptions(descriptions);

        final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
        final String json = writer.writeValueAsString(newCategory);

        HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

        ResponseEntity response = testRestTemplate.postForEntity("/api/v1/private/category", entity, PersistableCategory.class);
        PersistableCategory cat = (PersistableCategory) response.getBody();
        assertThat(response.getStatusCode(), is(CREATED));
        assertNotNull(cat.getId());

    }
    
    /**
    @Test
    public void putCategory() throws Exception {

        //create
        PersistableCategory newCategory = new PersistableCategory();
        newCategory.setCode("angular");
        newCategory.setSortOrder(1);
        newCategory.setVisible(true);
        newCategory.setDepth(4);


        CategoryDescription description = new CategoryDescription();
        description.setLanguage("en");
        description.setName("angular");
        description.setFriendlyUrl("angular");
        description.setTitle("angular");

        List<CategoryDescription> descriptions = new ArrayList<>();
        descriptions.add(description);

        newCategory.setDescriptions(descriptions);

        final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
        final String json = writer.writeValueAsString(newCategory);

        final HttpEntity<String> entity = new HttpEntity<>(json, getHeader());
        //create category
        final ResponseEntity response = testRestTemplate.postForEntity("/api/v1/private/category", entity, PersistableCategory.class);
        final PersistableCategory cat = (PersistableCategory) response.getBody();
        assertThat(response.getStatusCode(), is(CREATED));
        assertNotNull(cat.getId());
        
        HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());


        
        final ResponseEntity<ReadableCategory> readableQuery = testRestTemplate.exchange(String.format("/api/v1//category/" +  description.getFriendlyUrl()), HttpMethod.GET,
            httpEntity, ReadableCategory.class);
        
        assertThat(readableQuery.getStatusCode(), is(OK));
        
        ReadableCategory readableCategory = readableQuery.getBody();
        
        newCategory = new PersistableCategory();
        newCategory.setCode("angular");
        newCategory.setVisible(true);
        newCategory.setDepth(4);
        newCategory.setSortOrder(2);
        description = new CategoryDescription();
        description.setLanguage("en");
        description.setName("angular");
        description.setFriendlyUrl("angular");
        description.setTitle("angular");

        descriptions = new ArrayList<>();
        descriptions.add(description);

        newCategory.setDescriptions(descriptions);

        
        HttpEntity<PersistableCategory> requestUpdate = new HttpEntity<>(newCategory, getHeader());
        
        ResponseEntity resp = testRestTemplate.exchange("/api/v1/private/category/" + cat.getId(), HttpMethod.PUT,   requestUpdate, Void.class);
        assertThat(resp.getStatusCode(), is(OK));
        
        //update

    }
     **/

    @Test
    public void postComplexCategory() throws Exception {

        /** Dining room **/
        final PersistableCategory dining = new PersistableCategory();
        dining.setCode("diningroom");
        dining.setSortOrder(0);
        dining.setVisible(true);

        CategoryDescription endescription = new CategoryDescription();
        endescription.setLanguage("en");
        endescription.setName("Dining room");
        endescription.setFriendlyUrl("dining-room");
        endescription.setTitle("Dining room");

        CategoryDescription frdescription = new CategoryDescription();
        frdescription.setLanguage("fr");
        frdescription.setName("Salle à manger");
        frdescription.setFriendlyUrl("salle-a-manger");
        frdescription.setTitle("Salle à manger");

        List<CategoryDescription> descriptions = new ArrayList<>();
        descriptions.add(endescription);
        descriptions.add(frdescription);

        dining.setDescriptions(descriptions);

        final Category diningParent = new Category();
        diningParent.setCode(dining.getCode());

        /** armoire **/
        final PersistableCategory armoire = new PersistableCategory();
        armoire.setCode("armoire");
        armoire.setSortOrder(1);
        armoire.setVisible(true);

        armoire.setParent(diningParent);

        endescription = new CategoryDescription();
        endescription.setLanguage("en");
        endescription.setName("Armoires");
        endescription.setFriendlyUrl("armoires");
        endescription.setTitle("Armoires");

        frdescription = new CategoryDescription();
        frdescription.setLanguage("fr");
        frdescription.setName("Armoire");
        frdescription.setFriendlyUrl("armoires");
        frdescription.setTitle("Armoires");

        descriptions = new ArrayList<>();
        descriptions.add(endescription);
        descriptions.add(frdescription);

        armoire.setDescriptions(descriptions);
        dining.getChildren().add(armoire);

        /** benches **/
        final PersistableCategory bench = new PersistableCategory();
        bench.setCode("bench");
        bench.setSortOrder(4);
        bench.setVisible(true);

        bench.setParent(diningParent);

        endescription = new CategoryDescription();
        endescription.setLanguage("en");
        endescription.setName("Benches");
        endescription.setFriendlyUrl("benches");
        endescription.setTitle("Benches");

        frdescription = new CategoryDescription();
        frdescription.setLanguage("fr");
        frdescription.setName("Bancs");
        frdescription.setFriendlyUrl("bancs");
        frdescription.setTitle("Bancs");

        descriptions = new ArrayList<>();
        descriptions.add(endescription);
        descriptions.add(frdescription);

        bench.setDescriptions(descriptions);
        dining.getChildren().add(bench);

        /** Living room **/
        final PersistableCategory living = new PersistableCategory();
        living.setCode("livingroom");
        living.setSortOrder(2);
        living.setVisible(true);

        endescription = new CategoryDescription();
        endescription.setLanguage("en");
        endescription.setName("Living room");
        endescription.setFriendlyUrl("living-room");
        endescription.setTitle("Living room");

        frdescription = new CategoryDescription();
        frdescription.setLanguage("fr");
        frdescription.setName("Salon");
        frdescription.setFriendlyUrl("salon");
        frdescription.setTitle("Salon");

        descriptions = new ArrayList<>();
        descriptions.add(endescription);
        descriptions.add(frdescription);

        living.setDescriptions(descriptions);

        /** lounge **/

        final PersistableCategory lounge = new PersistableCategory();
        lounge.setCode("lounge");
        lounge.setSortOrder(3);
        lounge.setVisible(true);

        final Category livingParent = living;
        lounge.setParent(livingParent);

        endescription = new CategoryDescription();
        endescription.setLanguage("en");
        endescription.setName("Lounge");
        endescription.setFriendlyUrl("lounge");
        endescription.setTitle("Lounge");

        frdescription = new CategoryDescription();
        frdescription.setLanguage("fr");
        frdescription.setName("Divan");
        frdescription.setFriendlyUrl("divan");
        frdescription.setTitle("Divan");

        descriptions = new ArrayList<>();
        descriptions.add(endescription);
        descriptions.add(frdescription);

        lounge.setDescriptions(descriptions);
        living.getChildren().add(lounge);

        final ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
        final String json = writer.writeValueAsString(dining);

        //System.out.println(json);

        final HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

        final int sizeBefore = testRestTemplate.exchange(String.format("/api/v1/category"), HttpMethod.GET,
                new HttpEntity<>(getHeader()), ReadableCategoryList.class).getBody().getCategories().size();

        final ResponseEntity response = testRestTemplate.postForEntity("/api/v1/private/category", entity, PersistableCategory.class);

        final PersistableCategory cat = (PersistableCategory) response.getBody();
        assertThat(response.getStatusCode(), is(CREATED));
        assertNotNull(cat.getId());



    }

    @Test
    public void deleteCategory() throws Exception {

        final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());

        testRestTemplate.exchange("/services/DEFAULT/category/100", HttpMethod.DELETE, httpEntity, Category.class);
    }

    @Test
    public void manufacturerForItemsInCategory() throws Exception {
      
      ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
      
      //create first manufacturer
      PersistableManufacturer m1 = super.manufacturer("BRAND1");   
      
      String json = writer.writeValueAsString(m1);
      HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

      @SuppressWarnings("rawtypes")
      ResponseEntity response = testRestTemplate.postForEntity("/api/v1/private/manufacturer", entity, PersistableManufacturer.class);
      assertThat(response.getStatusCode(), is(CREATED));

      //create second manufacturer
      PersistableManufacturer m2 = super.manufacturer("BRAND2");
      json = writer.writeValueAsString(m2);
      entity = new HttpEntity<>(json, getHeader());

      response = testRestTemplate.postForEntity("/api/v1/private/manufacturer", entity, PersistableManufacturer.class);
      assertThat(response.getStatusCode(), is(CREATED));
      
      //create category
      PersistableCategory category = super.category("TEST");
      Category cat = new Category();//to be used in product
      cat.setCode("TEST");
      
      json = writer.writeValueAsString(category);
      entity = new HttpEntity<>(json, getHeader());

      @SuppressWarnings("rawtypes")
      ResponseEntity categoryResponse = testRestTemplate.postForEntity("/api/v1/private/category", entity, PersistableCategory.class);
      assertThat(categoryResponse.getStatusCode(), is(CREATED));
      final PersistableCategory persistable = (PersistableCategory) categoryResponse.getBody();
      
      Long id = persistable.getId();
      
      //create first item
      
      PersistableProduct product1 = super.product("PRODUCT1");
      product1.getCategories().add(cat);
      
      
      ProductSpecification specifications = new ProductSpecification();
      specifications.setManufacturer("BRAND1");
      product1.setProductSpecifications(specifications);
      
      json = writer.writeValueAsString(product1);
      entity = new HttpEntity<>(json, getHeader());

      response = testRestTemplate.postForEntity("/api/v1/private/product?store=" + Constants.DEFAULT_STORE, entity, PersistableProduct.class);
      assertThat(response.getStatusCode(), is(CREATED));
            
      //create second item      
      
      PersistableProduct product2 = super.product("PRODUCT2");
      product2.getCategories().add(cat);
      
      
      specifications = new ProductSpecification();
      specifications.setManufacturer("BRAND2");
      product2.setProductSpecifications(specifications);
      
      json = writer.writeValueAsString(product2);
      entity = new HttpEntity<>(json, getHeader());

      response = testRestTemplate.postForEntity("/api/v1/private/product?store=" + Constants.DEFAULT_STORE, entity, PersistableProduct.class);
      assertThat(response.getStatusCode(), is(CREATED));
      
      entity = new HttpEntity<>(getHeader());
            
      //get manufacturers in category
      @SuppressWarnings("rawtypes")
      ResponseEntity<List> manufacturers = testRestTemplate.exchange(String.format("/api/v1/category/" + id + "/manufacturer"), HttpMethod.GET, entity, List.class);  
      assertThat(manufacturers.getStatusCode(), is(OK));
      
      @SuppressWarnings("unchecked")
      List<ReadableManufacturer> manufacturerList = manufacturers.getBody();

      
      //assertFalse(manufacturerList.isEmpty());
      

      
      
    }
    
    
    /**
     * Test category by name
     * @throws Exception
     */

     /**
    
    @Test
    public void getByCategoryFriendlyUrl() throws Exception {
    	
    	ObjectWriter writer = new ObjectMapper().writer().withDefaultPrettyPrinter();
    	
    	String categoryName = "running-shoes";
    	String categoryCode = "runningshoes";
    	PersistableCategory category = category(categoryCode,categoryName);
        final String json = writer.writeValueAsString(category);

        HttpEntity<String> entity = new HttpEntity<>(json, getHeader());

        ResponseEntity response = testRestTemplate.postForEntity("/api/v1/private/category", entity, PersistableCategory.class);
        PersistableCategory cat = (PersistableCategory) response.getBody();
        assertThat(response.getStatusCode(), is(CREATED));
        assertNotNull(cat.getId());

        final ResponseEntity<ReadableCategory> readResponse = testRestTemplate.exchange(String.format("/api/v1/category/" + categoryName), HttpMethod.GET,
        		entity, ReadableCategory.class);
        if (readResponse.getStatusCode() != HttpStatus.OK) {
            throw new Exception(response.toString());
        } else {
            final ReadableCategory categ = readResponse.getBody();
            assertNotNull(readResponse);
            assertTrue(categoryCode.equals(categ.getCode()));
        }
    }
        **/

}


```
