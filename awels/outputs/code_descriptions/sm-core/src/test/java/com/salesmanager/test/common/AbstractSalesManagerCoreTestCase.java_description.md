# AbstractSalesManagerCoreTestCase.java

## Review

## 1. Summary  

**Purpose**  
`AbstractSalesManagerCoreTestCase` is a base test class that bootstraps the **Sales‑Manager** core services for integration testing. It loads the full Spring context (via `@SpringBootTest`), injects all domain‑layer services, and guarantees that the database is populated with a known test seed (`CONTEXT_LOAD_NAME = "TEST"`).  

**Key Components**  

| Component | Role |
|-----------|------|
| `@SpringBootTest(classes=ConfigurationTest.class)` | Loads the application context for the test. |
| `@RunWith(SpringRunner.class)` | Enables Spring’s JUnit integration. |
| `@Ignore` | Prevents JUnit from running any test classes that extend this one (useful when the suite is optional or too heavy). |
| `InitializationDatabase` | Service that checks if the DB is empty and populates it from a seed file. |
| `@Inject` fields | All domain services required by derived tests. |
| `init()` & `close()` | Lifecycle hooks executed before/after each test method. |

**Design Patterns & Frameworks**  

* **Dependency Injection** – Spring injects services into the test class.  
* **Template Method** – The base class provides the common `init()`/`close()` workflow, while concrete test classes implement specific tests.  
* **Factory/Configuration** – `ConfigurationTest` supplies the Spring beans needed for testing.  
* **JUnit 4** – The class uses JUnit 4 annotations (`@Before`, `@After`, `@Ignore`).  

---

## 2. Detailed Description  

### Flow of Execution  

1. **Test Class Loading**  
   * When a concrete test class extends `AbstractSalesManagerCoreTestCase`, the JUnit runner (`SpringRunner`) starts.  
   * `@SpringBootTest` triggers a full application context load using `ConfigurationTest`.  

2. **Dependency Injection**  
   * Spring injects all services annotated with `@Inject` (Spring’s equivalent of `@Autowired`).  

3. **Before Each Test (`@Before`)**  
   * `init()` is called.  
   * It checks `initializationDatabase.isEmpty()`.  
   * If the DB is empty, `populate()` is executed, which calls  
     `initializationDatabase.populate("TEST")`.  
     This loads seed data (categories, products, currencies, etc.) into the database.  

4. **Test Execution**  
   * The derived test class runs its test methods, using the injected services.  

5. **After Each Test (`@After`)**  
   * `close()` is invoked.  The current implementation is empty, so no cleanup occurs.  

6. **Termination**  
   * After all tests finish, the Spring context is closed by JUnit.  

### Assumptions & Constraints  

| Item | Assumption | Impact |
|------|------------|--------|
| Database state | Empty before the first test run. | `initializeDatabase.isEmpty()` will correctly detect this. |
| Test isolation | Tests do not modify global state. | If a test changes the DB, subsequent tests may see the mutated state. |
| Transactionality | No explicit rollback. | Side‑effects persist unless handled by the services. |
| Thread safety | Single‑threaded test execution. | Parallel test execution could cause race conditions. |
| `ConfigurationTest` | Provides a complete, consistent Spring configuration. | Any missing bean will cause context startup failure. |

### Architecture & Design Choices  

* **Full Context Integration Test** – The use of `@SpringBootTest` ensures that all services are wired exactly as in production, giving high confidence that the business layer behaves correctly.  
* **Centralized Data Seeding** – A single `InitializationDatabase` service is responsible for seeding, which keeps test data consistent across all tests.  
* **No Transaction Management** – The class relies on the underlying services to handle transactions. For pure integration tests this is acceptable, but it means test data persists across tests.  
* **JUnit 4** – The code predates JUnit 5, which offers more modern lifecycle annotations (`@BeforeEach`, `@AfterEach`, `@TestInstance`) and better extension points.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `init()` (`@Before`) | Called before each test method. Checks whether the DB needs seeding. | None | `void` | May call `populate()`; no other side effects. |
| `close()` (`@After`) | Called after each test method. Currently a placeholder. | None | `void` | None. |
| `populate()` | Private helper that triggers DB seeding using the `InitializationDatabase` service. | None | `void` | Calls `initializationDatabase.populate("TEST")`. |

### Notes on Reusable/Utility Methods  

* The `populate()` method is deliberately private because it is only used internally during initialization.  
* Constants (`CAD_CURRENCY_CODE`, `USD_CURRENCY_CODE`, `ENGLISH_LANGUAGE_CODE`, `FRENCH_LANGUAGE_CODE`) are `protected static` and can be reused by subclasses to refer to common test data.  

---

## 4. Dependencies  

| Dependency | Type | Usage |
|------------|------|-------|
| `javax.inject.Inject` | JSR‑330 (standard) | Annotation for DI. |
| `org.junit.*` | JUnit 4 (standard) | Test lifecycle annotations (`@Before`, `@After`, `@Ignore`). |
| `org.springframework.*` | Spring Framework (third‑party) | Test runner (`SpringRunner`), context loading (`@SpringBootTest`), and DI. |
| `com.salesmanager.*` | Application code (third‑party) | All domain services (`ProductService`, `OrderService`, etc.) and `InitializationDatabase`. |
| `com.salesmanager.test.configuration.ConfigurationTest` | Application test configuration | Supplies the bean definitions required for tests. |

*No platform‑specific or exotic dependencies are present; the code runs on any JVM that supports JUnit 4 and Spring.*

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Test Isolation** – Because `close()` does nothing, data created or modified in a test persists for subsequent tests. If tests depend on a clean state, they will interfere with each other.  
2. **Parallel Execution** – Running tests in parallel (e.g., with Maven’s surefire or Gradle’s test task) will share the same database instance, leading to race conditions.  
3. **Database Schema Changes** – If the seed data (`"TEST"`) no longer matches the updated schema, `populate()` may throw exceptions, causing all tests to fail.  
4. **Heavy Context Load** – `@SpringBootTest` loads the entire application context, which can be slow.  
5. **@Ignore Annotation** – This base class is annotated with `@Ignore`, meaning any subclass will be skipped unless the annotation is overridden. This may hide missing tests or be a reminder that integration tests should be run manually.

### Suggested Improvements  

| Area | Recommendation | Rationale |
|------|----------------|-----------|
| **Test Lifecycle** | Switch to JUnit 5 (`@BeforeEach`, `@AfterEach`) and use `@TestInstance(Lifecycle.PER_CLASS)` if a shared context is desired. | Cleaner API, better extension points. |
| **Transaction Management** | Annotate the class with `@Transactional` and configure `@Rollback(true)` on each test method. | Guarantees database rollback, ensuring isolation. |
| **Cleanup** | Implement `close()` to delete the seeded data or reset the database. | Prevents state leakage between tests. |
| **Parallel Safety** | Use an in‑memory database (e.g., H2) per test or leverage Spring’s `@DirtiesContext` to reload context between tests. | Avoids concurrency issues. |
| **Removing @Ignore** | Delete or comment out `@Ignore` unless integration tests are intentionally optional. | Ensures tests run as part of the normal build. |
| **Seeding Strategy** | Replace `initializationDatabase.populate()` with a dedicated TestEntityManager or fixture library (e.g., DBUnit, TestContainers). | More control over test data and easier maintenance. |
| **Logging & Diagnostics** | Add logging in `init()` and `populate()` to trace seed execution. | Helps diagnose test failures related to data loading. |
| **Configuration** | Externalize the seed name (`CONTEXT_LOAD_NAME`) to a system property or test property file. | Enables varying seeds per environment. |

### Future Enhancements  

1. **Parameterized Tests** – Use JUnit 5’s parameterized tests to run the same test logic against different locales or currencies.  
2. **Mocking** – Where possible, replace heavy services with mocks to reduce context load.  
3. **Test Containers** – Spin up a Dockerised database per test run to guarantee a pristine environment.  
4. **Metrics** – Capture test execution time and database query counts to identify performance regressions.  

---

### Bottom Line  

`AbstractSalesManagerCoreTestCase` is a well‑structured, dependency‑injected base class that sets up the Sales‑Manager core for integration testing. Its main strengths lie in centralizing data seeding and providing ready‑to‑use services for derived tests. However, its current lack of transaction handling and cleanup, coupled with the `@Ignore` annotation, limits its effectiveness in a CI pipeline. By adopting some of the improvements above, the test suite can become more robust, isolated, and maintainable.

## Code Critique



## Code Preview

```java
/**
 * This application is maintained by CSTI consulting (www.csticonsulting.com).
 * Licensed under LGPL - Feel free to use it and modify it to your needs !
 *
 */
package com.salesmanager.test.common;

import javax.inject.Inject;

import org.junit.After;
import org.junit.Before;
import org.junit.Ignore;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.category.CategoryService;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.catalog.product.attribute.ProductAttributeService;
import com.salesmanager.core.business.services.catalog.product.attribute.ProductOptionService;
import com.salesmanager.core.business.services.catalog.product.attribute.ProductOptionSetService;
import com.salesmanager.core.business.services.catalog.product.attribute.ProductOptionValueService;
import com.salesmanager.core.business.services.catalog.product.availability.ProductAvailabilityService;
import com.salesmanager.core.business.services.catalog.product.image.ProductImageService;
import com.salesmanager.core.business.services.catalog.product.manufacturer.ManufacturerService;
import com.salesmanager.core.business.services.catalog.product.price.ProductPriceService;
import com.salesmanager.core.business.services.catalog.product.relationship.ProductRelationshipService;
import com.salesmanager.core.business.services.catalog.product.review.ProductReviewService;
import com.salesmanager.core.business.services.catalog.product.type.ProductTypeService;
import com.salesmanager.core.business.services.customer.CustomerService;
import com.salesmanager.core.business.services.customer.attribute.CustomerOptionService;
import com.salesmanager.core.business.services.customer.attribute.CustomerOptionSetService;
import com.salesmanager.core.business.services.customer.attribute.CustomerOptionValueService;
import com.salesmanager.core.business.services.merchant.MerchantStoreService;
import com.salesmanager.core.business.services.order.OrderService;
import com.salesmanager.core.business.services.payments.PaymentService;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.currency.CurrencyService;
import com.salesmanager.core.business.services.reference.init.InitializationDatabase;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.business.services.reference.zone.ZoneService;
import com.salesmanager.core.business.services.shoppingcart.ShoppingCartService;
import com.salesmanager.core.business.services.system.EmailService;
import com.salesmanager.test.configuration.ConfigurationTest;


/**
 * @author c.samson
 *
 */

@RunWith(SpringRunner.class)
@SpringBootTest(classes=ConfigurationTest.class)
@Ignore
public class AbstractSalesManagerCoreTestCase {
	
	private static final String CONTEXT_LOAD_NAME = "TEST";

	
	
	protected static String CAD_CURRENCY_CODE = "CAD";
	protected static String USD_CURRENCY_CODE = "USD";
	
	protected static String ENGLISH_LANGUAGE_CODE = "en";
	protected static String FRENCH_LANGUAGE_CODE = "fr";
	
	@Inject
	protected InitializationDatabase   initializationDatabase;
	
	@Inject
	protected ProductService productService;
	
	@Inject
	protected PricingService pricingService;
	
	@Inject
	protected ProductPriceService productPriceService;
	
	@Inject
	protected ProductAttributeService productAttributeService;
	
	@Inject
	protected ProductOptionService productOptionService;
	
	@Inject
	protected ProductOptionSetService productOptionSetService;
	
	@Inject
	protected ProductOptionValueService productOptionValueService;
	
	@Inject
	protected ProductAvailabilityService productAvailabilityService;
	
	@Inject
	protected ProductReviewService productReviewService;
	
	@Inject
	protected ProductImageService productImageService;
	
	@Inject
	protected ProductRelationshipService productRelationshipService;
	
	@Inject
	protected CategoryService categoryService;
	
	@Inject
	protected MerchantStoreService merchantService;
	
	@Inject
	protected ProductTypeService productTypeService;
	
	@Inject
	protected LanguageService languageService;
	
	@Inject
	protected CountryService countryService;
	
	@Inject
	protected CurrencyService currencyService;
	
	@Inject
	protected ManufacturerService manufacturerService;
	
	@Inject
	protected ZoneService zoneService;
	
	@Inject
	protected CustomerService customerService;
	
	@Inject
	protected CustomerOptionService customerOptionService;
	
	@Inject
	protected CustomerOptionValueService customerOptionValueService;
	
	@Inject
	protected CustomerOptionSetService customerOptionSetService;
	
	@Inject
	protected OrderService orderService;
	
	@Inject
	protected PaymentService paymentService;
	
	@Inject
	protected ShoppingCartService shoppingCartService;
	
	@Inject
	protected EmailService emailService;
	
	@Before
	public void init() throws ServiceException {
		if(initializationDatabase.isEmpty()) {
		  populate();
		}

	}
	
	@After
	public void close() throws ServiceException {

	}
	
	private void populate() throws ServiceException {
		initializationDatabase.populate(CONTEXT_LOAD_NAME);
	}

}



```
