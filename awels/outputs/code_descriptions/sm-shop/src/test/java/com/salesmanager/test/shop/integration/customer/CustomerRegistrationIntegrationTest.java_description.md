# CustomerRegistrationIntegrationTest.java

## Review

## 1. Summary  
The file contains a **Spring Boot integration test** that verifies the end‑to‑end workflow for registering a new customer and subsequently logging in.  
Key aspects:  

| Component | Role |
|-----------|------|
| `PersistableCustomer` | DTO that represents a customer to be persisted. |
| `Address` | DTO for billing address information. |
| `AuthenticationRequest` / `AuthenticationResponse` | DTOs for the login API. |
| `testRestTemplate` | Provided by `ServicesTestSupport`; used to issue HTTP calls against the running test server. |
| `ShopApplication` | Main Spring Boot application class used to bootstrap the context. |

The test exercises the `/api/v1/customer/register` endpoint, checks that a successful response (HTTP 200) is returned, and then exercises `/api/v1/customer/login` to confirm that the newly created customer can authenticate.  
The design follows typical Spring Boot testing conventions, using `@SpringBootTest` with a random port, `@RunWith(SpringRunner.class)`, and Hamcrest assertions for readability.

---

## 2. Detailed Description  

### Execution Flow  
1. **Test Setup** – `@SpringBootTest` launches the full application context on a random port; `ServicesTestSupport` presumably provides the `testRestTemplate` and common headers (e.g., JSON `Content-Type`).  
2. **Customer DTO Construction** –  
   - Email, password, gender, and language are set.  
   - A billing `Address` is populated with first/last name, and country (`"BE"`).  
   - The store code is set to the default (`Constants.DEFAULT_STORE`).  
3. **POST /api/v1/customer/register** – The `PersistableCustomer` is wrapped in an `HttpEntity` with headers and sent to the registration endpoint.  
4. **Assertion** – The response status must be `200 OK`.  
5. **POST /api/v1/customer/login** – A new `AuthenticationRequest` (email & password) is sent to the login endpoint.  
6. **Assertion** – The login response must also be `200 OK`, and a non‑null JWT token is expected in the body.  

### Assumptions & Constraints  
- The test assumes that the application is reachable on a random port and that all required services (e.g., user repository, authentication provider) are correctly wired.  
- No cleanup logic is present; subsequent test runs will create duplicate customers unless the database is reset between tests.  
- The test does not validate the response payload beyond the status code and token existence; deeper verification (e.g., field values, audit logs) is omitted.  

### Architecture & Design Choices  
- **Full Integration Test**: Using `@SpringBootTest` ensures that the HTTP layer, controller, service, and persistence layers are all exercised.  
- **DTO‑Centric**: The test constructs business‑level DTOs rather than raw JSON, keeping the code type‑safe.  
- **Minimal Dependencies**: Only Spring Boot test facilities and Hamcrest are used for assertions.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `registerCustomer()` | Test method that verifies customer registration and login. | None (constructs its own DTOs) | None (asserts internally) | None; may create a new customer record in the test database. |

*Note*: The method is the sole public test entry point. All other logic resides in inherited helpers (`getHeader()`, `testRestTemplate`).

---

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| `org.springframework.boot:spring-boot-starter-test` | Third‑party | Provides JUnit, Spring test context, `MockMvc`, `TestRestTemplate`, and Hamcrest. |
| `org.springframework.boot:spring-boot-starter-web` | Third‑party | Required for REST controllers and HTTP clients. |
| `org.junit.jupiter:junit-jupiter-api` | Third‑party | JUnit 5 (or 4 if using the older runner) for test lifecycle. |
| `org.springframework.security` (implied) | Third‑party | Handles authentication; not directly referenced in the test but required for the login endpoint. |
| `com.salesmanager` packages | Project internal | Application code under test (controllers, services, DTOs). |

No platform‑specific dependencies are used; the test is portable across JVM environments.

---

## 5. Additional Notes  

### Strengths  
- **Readability**: The test clearly shows the flow: build customer → register → login.  
- **Type Safety**: Using DTOs prevents JSON syntax errors.  
- **Simplicity**: Minimal setup code makes the test easy to maintain.

### Potential Issues & Edge Cases  
1. **Database State Pollution** – Repeated test runs create duplicate customers (`customer1@test.com`). Consider resetting the database between runs or using unique emails (e.g., UUID suffix).  
2. **Hardcoded Values** – Email, password, and address fields are hardcoded; changes to the registration API (e.g., required fields) may break the test without clear failure signals.  
3. **Limited Validation** – Only status codes and token existence are verified. Unexpected validation errors (e.g., email format) would not surface unless they result in a non‑200 status.  
4. **Concurrency** – Running tests in parallel could cause race conditions when inserting the same email.  

### Suggested Enhancements  
- **Parameterized Tests**: Use JUnit’s `@ParameterizedTest` to run registration with various data sets (missing fields, invalid email).  
- **Database Cleanup**: Override `@Transactional` with `rollback` or use a dedicated test database that is recreated before each test.  
- **Response Payload Checks**: Assert that the returned `PersistableCustomer` contains the expected fields, and that the authentication response contains a valid JWT format.  
- **Error Path Testing**: Add tests for failure scenarios (e.g., duplicate email, weak password) to ensure proper error handling.  
- **Security**: Verify that the JWT token is signed and contains expected claims (user ID, roles).  

Overall, the integration test is concise and serves its primary purpose, but expanding its coverage and isolating test data would improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.customer;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.OK;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.http.HttpEntity;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.model.customer.CustomerGender;
import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.customer.PersistableCustomer;
import com.salesmanager.shop.model.customer.address.Address;
import com.salesmanager.shop.store.security.AuthenticationRequest;
import com.salesmanager.shop.store.security.AuthenticationResponse;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
public class CustomerRegistrationIntegrationTest extends ServicesTestSupport {

    @Test
    public void registerCustomer() {
        
      
        final PersistableCustomer testCustomer = new PersistableCustomer();
        testCustomer.setEmailAddress("customer1@test.com");
        testCustomer.setPassword("clear123");
        testCustomer.setGender(CustomerGender.M.name());
        testCustomer.setLanguage("en");
        final Address billing = new Address();
        billing.setFirstName("customer1");
        billing.setLastName("ccstomer1");
        billing.setCountry("BE");
        testCustomer.setBilling(billing);
        testCustomer.setStoreCode(Constants.DEFAULT_STORE);
        final HttpEntity<PersistableCustomer> entity = new HttpEntity<>(testCustomer, getHeader());

        final ResponseEntity<PersistableCustomer> response = testRestTemplate.postForEntity("/api/v1/customer/register", entity, PersistableCustomer.class);
        assertThat(response.getStatusCode(), is(OK));

        // created customer can login

        final ResponseEntity<AuthenticationResponse> loginResponse = testRestTemplate.postForEntity("/api/v1/customer/login", new HttpEntity<>(new AuthenticationRequest("customer1@test.com", "clear123")),
                AuthenticationResponse.class);
        assertThat(loginResponse.getStatusCode(), is(OK));
        assertNotNull(loginResponse.getBody().getToken());

    }

}


```
