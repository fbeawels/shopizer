# MerchantStoreApiIntegrationTest.java

## Review

## 1. Summary  

**Purpose**  
The class `MerchantStoreApiIntegrationTest` provides end‑to‑end integration tests for the **Merchant Store** REST API exposed by the `ShopApplication`. It verifies that the API correctly handles retrieving the default store, creating a new store, and adding/removing a store logo.

**Key Components**  
| Component | Role |
|-----------|------|
| `@SpringBootTest` + `WebEnvironment.RANDOM_PORT` | Boots a full Spring context and starts an embedded servlet container on a random port for realistic HTTP interactions. |
| `TestRestTemplate` | Simplifies sending HTTP requests and receiving responses during tests. |
| `MerchantStore.DEFAULT_STORE` | Constant representing the default store identifier used by the API. |
| `PersistableAddress`, `PersistableMerchantStore`, `ReadableMerchantStore` | DTOs used to serialize/deserialize request/response payloads. |
| `ServicesTestSupport` | (Not shown) likely contains helper methods such as `getHeader()` that provide common request headers (e.g., authentication tokens). |

**Design Patterns / Libraries**  
*Spring Framework (Boot, MVC, Test)*  
*Junit 4 (`@RunWith(SpringRunner.class)`)  
*Hamcrest (`assertThat`, `is`) for fluent assertions.*  

---

## 2. Detailed Description  

### Execution Flow  

1. **Bootstrap**  
   - Spring loads the `ShopApplication` context.
   - An embedded HTTP server listens on a random port.
   - `TestRestTemplate` is injected and automatically configured to point to that port.

2. **Test Cases**  
   - `testGetDefaultStore()`  
     * Sends a `GET` to `/api/v1/store/default`.  
     * Asserts that the response status is 200 OK and the body is non‑null.

   - `testCreateStore()`  
     * Builds a `PersistableMerchantStore` with a nested `PersistableAddress`.  
     * Sends a `POST` to `/api/v1/private/store/`.  
     * Expects HTTP 200 OK (the API returns no body).

   - `testAddAndDeleteStoreLogo()`  
     * Uploads a logo file (`image.jpg`) via a multipart `POST`.  
     * Expects HTTP 201 CREATED.  
     * Then deletes the logo via a `DELETE` request.  
     * Expects HTTP 200 OK.

3. **Cleanup**  
   - The tests do not perform explicit cleanup. The embedded server shuts down automatically when the JVM exits.

### Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| `getHeader()` supplies a valid authentication header. | Tests will fail if the header is missing or malformed. |
| The embedded file `image.jpg` exists on the classpath. | File not found will cause an exception before the request is sent. |
| The API returns HTTP 200 for `POST /api/v1/private/store/` even when the store is successfully created. | If the API changes to return 201 or a body, the test will need adjustment. |
| No verification of side‑effects (e.g., store persistence, logo storage). | Tests only confirm HTTP status, not data integrity. |

### Architecture & Design Choices  

* **Integration‑Level Testing** – Using `SpringBootTest` with a real HTTP client ensures that all layers (controller, service, repository, security) are exercised together.  
* **DTO‑Only Tests** – The tests rely solely on DTOs (`Persistable*` and `Readable*`). They avoid any direct database or service layer interaction.  
* **Static Header Utility** – `getHeader()` abstracts header creation, centralizing authentication logic.  
* **Hard‑coded Strings** – Test values (`TEST_STORE_CODE`, `CURRENCY`, etc.) are constants, making the tests easy to read and modify.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return Type | Side‑Effects |
|--------|---------|------------|-------------|--------------|
| `testGetDefaultStore()` | Verify that the default store endpoint returns a non‑null store. | none | `void` | Throws `Exception` on non‑200 response. |
| `testCreateStore()` | Create a new store and verify successful creation. | none | `void` | Sends a POST; expects 200 OK. |
| `testAddAndDeleteStoreLogo()` | Upload a logo for the default store and then delete it. | none | `void` | Sends POST/DELETE; verifies status codes. |

These are JUnit test methods; they do not return values but use assertions to indicate success or failure. All methods rely on the injected `TestRestTemplate` and the `getHeader()` helper.

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `org.springframework.boot.test.context.SpringBootTest` | Third‑party (Spring Boot) | Bootstraps the application context for integration tests. |
| `org.springframework.boot.test.web.client.TestRestTemplate` | Third‑party | Provides a convenient REST client pre‑configured with the embedded server URL. |
| `org.springframework.util.LinkedMultiValueMap` | Third‑party | Represents multipart form data. |
| `org.hamcrest.MatcherAssert` / `org.hamcrest.Matchers` | Third‑party | Fluent assertion utilities. |
| `org.junit.Assert` | Third‑party (JUnit) | Basic assertions (`assertNotNull`). |
| `org.junit.Test` | Third‑party (JUnit) | Declares test methods. |
| `org.springframework.test.context.junit4.SpringRunner` | Third‑party | JUnit runner that integrates with Spring. |
| `org.springframework.core.io.ClassPathResource` | Third‑party | Loads resources from the classpath (logo image). |
| `com.salesmanager.core.model.merchant.MerchantStore` | Internal | Holds constants like `DEFAULT_STORE`. |
| DTOs (`PersistableAddress`, `PersistableMerchantStore`, `ReadableMerchantStore`) | Internal | Transfer objects for API payloads. |
| `ServicesTestSupport` | Internal | Provides `getHeader()` and likely other helpers. |

All dependencies are standard for a Spring Boot application; no platform‑specific libraries are used.

---

## 5. Additional Notes  

### Strengths  
* **Full integration coverage** – Tests traverse the entire stack, catching issues that unit tests might miss.  
* **Clear separation of concerns** – Each test focuses on a single endpoint/action.  
* **Reusability of helpers** – `getHeader()` centralises authentication handling.  

### Areas for Improvement  
1. **Data Verification**  
   * `testCreateStore()` only checks the HTTP status. It would be more robust to retrieve the created store (via `GET`) and assert that the persisted fields match the request.  
   * `testAddAndDeleteStoreLogo()` could verify the presence of the logo after upload (e.g., by retrieving the store’s marketing resources).

2. **Cleanup / Idempotence**  
   * The `testCreateStore()` test leaves a new store in the system. Subsequent test runs may fail due to duplicate codes. Implement a teardown that deletes the store or use a unique random code per run.  
   * Consider using `@Transactional` with `@Rollback` if the underlying repository supports it.

3. **Error Handling**  
   * The tests throw generic `Exception` when the response status is not OK. Using JUnit’s assertion helpers (`assertEquals`, `assertThat`) would provide clearer failure messages and avoid unnecessary exception throwing.

4. **Magic Strings / Magic Numbers**  
   * Hard‑coded paths like `"/api/v1/store/" + MerchantStore.DEFAULT_STORE` could be replaced with a constants class or `UriComponentsBuilder` to reduce duplication and improve readability.

5. **Use of JUnit 5**  
   * Modern Spring Boot projects often use JUnit 5 (`@ExtendWith(SpringExtension.class)`). Migrating could enable more features (e.g., parameterized tests).

6. **Parameterization & Randomization**  
   * Using a UUID for `TEST_STORE_CODE` in each run prevents collisions and makes tests independent.

### Future Enhancements  
* **Parameterized Tests** – Verify the API with various languages, currencies, and invalid inputs.  
* **Contract Testing** – Generate API contracts (e.g., using Pact) to ensure the service meets consumer expectations.  
* **Performance Metrics** – Measure response times for each endpoint.  
* **Security Tests** – Verify that unauthorized requests are properly rejected.  

Overall, the test suite provides a solid foundation for ensuring the Merchant Store API behaves as expected, but adding deeper assertions and cleanup logic would increase reliability and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.store;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.is;
import static org.junit.Assert.assertNotNull;
import java.util.Arrays;
import javax.inject.Inject;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.util.LinkedMultiValueMap;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.references.PersistableAddress;
import com.salesmanager.shop.model.store.PersistableMerchantStore;
import com.salesmanager.shop.model.store.ReadableMerchantStore;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
public class MerchantStoreApiIntegrationTest extends ServicesTestSupport {
  
  private static final String TEST_STORE_CODE = "test";
  private static final String CURRENCY = "CAD";
  private static final String DEFAULT_LANGUAGE = "en";

  @Inject
  private TestRestTemplate testRestTemplate;
  
  /**
   * Test get DEFAULT store
   * @throws Exception
   */
  @Test
  public void testGetDefaultStore() throws Exception {
      final HttpEntity<String> httpEntity = new HttpEntity<>(getHeader());

      final ResponseEntity<ReadableMerchantStore> response = testRestTemplate.exchange(String.format("/api/v1/store/" + MerchantStore.DEFAULT_STORE), HttpMethod.GET,
              httpEntity, ReadableMerchantStore.class);
      if (response.getStatusCode() != HttpStatus.OK) {
          throw new Exception(response.toString());
      } else {
          final ReadableMerchantStore store = response.getBody();
          assertNotNull(store);
      }
  }
  
  /**
   * Create a new store then delete it
   * @throws Exception
   */
  @Test
  public void testCreateStore() throws Exception {
      
      
      PersistableAddress address = new PersistableAddress();
      address.setAddress("121212 simple address");
      address.setPostalCode("12345");
      address.setCountry("US");
      address.setCity("FT LD");
      address.setStateProvince("FL");

      PersistableMerchantStore createdStore = new PersistableMerchantStore();
      createdStore.setCode(TEST_STORE_CODE);
      createdStore.setCurrency(CURRENCY);
      createdStore.setDefaultLanguage(DEFAULT_LANGUAGE);
      createdStore.setEmail("test@test.com");
      createdStore.setName(TEST_STORE_CODE);
      createdStore.setPhone("444-555-6666");
      createdStore.setSupportedLanguages(Arrays.asList(DEFAULT_LANGUAGE));
      createdStore.setAddress(address);
      
      final HttpEntity<PersistableMerchantStore> httpEntity = new HttpEntity<PersistableMerchantStore>(createdStore, getHeader());

      ResponseEntity<Void> response = testRestTemplate.exchange(String.format("/api/v1/private/store/"), HttpMethod.POST, httpEntity, Void.class);

      assertThat(response.getStatusCode(), is(HttpStatus.OK));

  }
  
  
  @Test
  public void testAddAndDeleteStoreLogo() {
      LinkedMultiValueMap<String, Object> parameters = new LinkedMultiValueMap<String, Object>();
      parameters.add("file", new org.springframework.core.io.ClassPathResource("image.jpg"));

      HttpHeaders headers = getHeader();
      headers.setContentType(MediaType.MULTIPART_FORM_DATA);

      HttpEntity<LinkedMultiValueMap<String, Object>> entity = new HttpEntity<LinkedMultiValueMap<String, Object>>(parameters, headers);

      ResponseEntity<Void> createResponse = testRestTemplate.exchange(String.format("/api/v1/private/store/" + MerchantStore.DEFAULT_STORE + "/marketing/logo"), HttpMethod.POST, entity, Void.class);

      // Expect Created
      assertThat(createResponse.getStatusCode(), is(HttpStatus.CREATED));
      
      // now remove logo
      HttpEntity<Void> deleteRequest = new HttpEntity<Void>(getHeader());
      
      ResponseEntity<Void> deleteResponse = testRestTemplate.exchange(String.format("/api/v1/private/store/" + MerchantStore.DEFAULT_STORE + "/marketing/logo"), HttpMethod.DELETE, deleteRequest, Void.class);

      // Expect Ok
      assertThat(deleteResponse.getStatusCode(), is(HttpStatus.OK));

  }

  
  
}



```
