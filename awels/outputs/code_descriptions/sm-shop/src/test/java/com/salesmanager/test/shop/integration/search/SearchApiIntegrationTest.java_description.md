# SearchApiIntegrationTest.java

## Review

## 1. Summary  

This file defines a **Spring Boot integration test** for the Shop application's search API.  
It uses the `TestRestTemplate` to exercise the REST endpoints that:

1. Persist a new product (`/api/v1/private/product`)  
2. Query the search service (`/api/v1/search`)  

The test is currently **ignored** (both the class and method are marked with `@Ignore`), indicating it is either incomplete or depends on an external resource (Elastic Search) that is not available in the test environment.  

Key components:  

| Component | Role |
|-----------|------|
| `@SpringBootTest` | Boots the whole application context for integration testing. |
| `TestRestTemplate` | Simplifies REST calls against the embedded test server. |
| `PersistableProduct` | Domain model used for product creation. |
| `SearchProductRequest / SearchProductList` | DTOs used to issue and receive search queries. |
| `ServicesTestSupport` | Base class that presumably supplies common test utilities (e.g., `product()` factory, `getHeader()`). |

The test demonstrates a simple *add‑then‑search* flow, asserting that the HTTP status for both operations is `201 CREATED`. No content assertions are performed beyond the status code.

---

## 2. Detailed Description  

### Execution Flow  

1. **Spring Context Loading**  
   * `@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)` starts the full application on a random port.  
   * `@RunWith(SpringRunner.class)` integrates Spring’s test runner.  
   * `@Ignore` at the class level disables the entire test suite until it is intentionally re‑enabled.

2. **Test Preparation**  
   * `@Autowired TestRestTemplate testRestTemplate` injects a ready‑to‑use REST client configured with the test server’s base URL.  
   * The helper method `super.product("TESTPRODUCT")` (inherited from `ServicesTestSupport`) presumably creates a `PersistableProduct` instance pre‑filled with minimal required fields.  
   * `getHeader()` (also inherited) probably returns an `HttpHeaders` object containing authentication and content‑type information.

3. **Product Creation**  
   * The test serialises the `PersistableProduct` to JSON and posts it to `/api/v1/private/product?store=DEFAULT_STORE`.  
   * It expects a `201 CREATED` response.  

4. **Search Request**  
   * A `SearchProductRequest` with a simple query string `"TEST"` is wrapped in an `HttpEntity` and POSTed to `/api/v1/search?store=DEFAULT_STORE`.  
   * Again, a `201 CREATED` response is asserted.

5. **Termination**  
   * No explicit cleanup is performed; Spring’s context shutdown will discard the persisted data.

### Assumptions & Constraints  

| Assumption | Explanation |
|------------|-------------|
| ElasticSearch is available | The comment indicates the test requires an ES server, which is likely why it is disabled. |
| `Constants.DEFAULT_STORE` is a valid store ID | The test hard‑codes the store parameter. |
| POST to `/search` returns 201 | Usually a search operation would return `200 OK`. The API contract must reflect this behaviour. |
| `getHeader()` supplies necessary auth | Without proper headers the endpoints would reject the request. |

### Design Choices  

* **Full Integration** – By starting the whole application, the test exercises real networking, serialization, and service layers.  
* **Ignored Test** – Demonstrates a pragmatic approach: keep the test in source but disable it until the required infrastructure (ES) is present.  
* **Minimal Assertions** – The focus is on HTTP status; content validation is omitted.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `searchItem()` | Integration test that creates a product then performs a search. | None | `void` | Executes REST calls; asserts HTTP status. |
| `super.product(String)` | (Inherited) Factory that builds a `PersistableProduct` with a given code/name. | `String` product code | `PersistableProduct` | None. |
| `getHeader()` | (Inherited) Builds HTTP headers for authenticated requests. | None | `HttpHeaders` | None. |

> **Note:** The test is annotated with both `@Test` (commented out) and `@Ignore`. The method body is effectively a **commented test** – it can be activated by removing the `@Ignore` and uncommenting the `@Test` annotation.

---

## 4. Dependencies  

| Library / Framework | Role | Type |
|---------------------|------|------|
| **Spring Boot** (`spring-boot-starter-test`, `spring-boot-starter-web`) | Provides test runner, context, REST client. | Third‑party |
| **JUnit 4** (`org.junit`) | Test framework. | Third‑party |
| **Hamcrest** (`org.hamcrest`) | Fluent assertions (`assertThat`). | Third‑party |
| **Spring Test** (`org.springframework.test`) | `SpringRunner`, context integration. | Third‑party |
| **TestRestTemplate** (`org.springframework.boot.test.web.client`) | Simplified REST calls in tests. | Part of Spring Boot |
| **Application code** (`com.salesmanager.*`) | Domain models, constants, controllers. | Internal |

*No explicit ElasticSearch client is used here; the test merely assumes ES is running externally.*

---

## 5. Additional Notes  

### Edge Cases & Missing Validations  

1. **Search Response Body** – The test only checks status; it does not verify that the created product appears in the search results.  
2. **Duplicate Product** – If the test is run multiple times without cleanup, it may create duplicate products or fail due to unique constraints.  
3. **HTTP Status Semantics** – Search normally returns `200 OK`. If the API really returns `201`, this should be documented; otherwise, the assertion may mask an error.  
4. **Parameterization** – Hard‑coding `Constants.DEFAULT_STORE` limits reusability. Consider using test parameters or a dynamic store ID.  

### Potential Enhancements  

| Idea | Benefit |
|------|---------|
| Enable the test with a conditional **@ConditionalOnProperty** that checks for ES availability. | Avoids manual toggling. |
| Add assertions on `searchResponse.getBody()` to confirm the expected product is present. | Increases test coverage. |
| Clean up created products in an `@After` method or use `@DirtiesContext` to reset state. | Prevents test pollution. |
| Refactor repeated code (product creation, search request) into helper methods. | Reduces duplication. |
| Use `@SpringBootTest(webEnvironment = WebEnvironment.MOCK)` with `MockMvc` for faster execution when ES is mocked. | Improves test speed. |

### Summary  

The integration test skeleton is solid and correctly wired into Spring’s testing framework. However, it currently serves as a placeholder rather than an active verification of the search feature. Activating it requires an operational ElasticSearch instance and a decision on the expected HTTP semantics for search. Once enabled, enriching the assertions and ensuring isolation will make it a valuable part of the regression suite.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.search;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.CREATED;

import org.junit.Ignore;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.catalog.SearchProductList;
import com.salesmanager.shop.model.catalog.SearchProductRequest;
import com.salesmanager.shop.model.catalog.product.product.PersistableProduct;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
@Ignore
public class SearchApiIntegrationTest extends ServicesTestSupport {

    @Autowired
    private TestRestTemplate testRestTemplate;



    /**
     * Add a product then search for it
     * This tests is disabled since it requires Elastic search server started
     *
     * @throws Exception
     */
    //@Test
    @Ignore
    public void searchItem() throws Exception {
    	
    	PersistableProduct product = super.product("TESTPRODUCT");
    	
        final HttpEntity<PersistableProduct> entity = new HttpEntity<>(product, getHeader());

        final ResponseEntity<PersistableProduct> response = testRestTemplate.postForEntity("/api/v1/private/product?store=" + Constants.DEFAULT_STORE, entity, PersistableProduct.class);
        assertThat(response.getStatusCode(), is(CREATED));
        
        SearchProductRequest searchRequest = new SearchProductRequest();
        searchRequest.setQuery("TEST");
        final HttpEntity<SearchProductRequest> searchEntity = new HttpEntity<>(searchRequest, getHeader());
        
        
        final ResponseEntity<SearchProductList> searchResponse = testRestTemplate.postForEntity("/api/v1/search?store=" + Constants.DEFAULT_STORE, searchEntity, SearchProductList.class);
        assertThat(searchResponse.getStatusCode(), is(CREATED));

    }


}


```
