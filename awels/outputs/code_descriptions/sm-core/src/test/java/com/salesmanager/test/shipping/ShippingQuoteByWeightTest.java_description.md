# ShippingQuoteByWeightTest.java

## Review

## 1. Summary

| Aspect | Observation |
|--------|-------------|
| **Purpose** | The class is a JUnit integration test that exercises a *weight‑based* shipping quote flow in the SalesManager core system. It creates a product, configures shipping settings, persists a customer and address data, and finally asserts that a shipping quote can be generated. |
| **Key Components** | *ShippingService*, *LanguageService*, *ProductService*, *ShippingConfiguration*, *CustomShippingQuotesConfiguration*, *IntegrationConfiguration*, *Customer*, *Billing*, *Delivery*, and the `ShippingProduct`/`ShippingQuote` objects. |
| **Design Patterns / Frameworks** | - Dependency Injection via `@Inject` (likely CDI or Spring).<br>- Use of the *Strategy* pattern for shipping modules (moduleCode “weightBased”).<br>- The test itself is a *black‑box integration test*; it relies heavily on the underlying persistence layer. |
| **Notable Libraries** | Spring’s `Assert`, JUnit (implicit via `@Test`/`@Ignore`), and the custom SalesManager core domain model. |

---

## 2. Detailed Description

### Execution Flow

1. **Environment Setup**  
   - Language, country, zone, and merchant store are fetched from services.  
   - The store’s postal code is manually set.

2. **Product Creation**  
   - A `Product` is created with basic dimensions and a weight of 8 kg.  
   - A `ProductDescription` is added.  
   - Availability and pricing are configured and persisted.

3. **Shipping Configuration**  
   - A `ShippingConfiguration` is created with shipping type, package type, and box dimensions.  
   - Supported countries are defined and persisted.  
   - A custom shipping module (`weightBased`) is configured with regions and weight brackets.  
   - The module is persisted via `ShippingService`.

4. **Customer & Address Setup**  
   - A `Customer` is created with billing and delivery addresses (Canada).  
   - The customer is persisted.

5. **Quote Calculation**  
   - A `ShippingProduct` is built from the product and its final price.  
   - `ShippingService.getShippingQuote` is invoked with the dummy cart ID, store, delivery, shipping products, and language.  
   - The test asserts that a non‑null quote is returned.

### Assumptions & Constraints

| Aspect | Assumption |
|--------|------------|
| **Persistence Layer** | All services interact with a real database (or an in‑memory DB). No mocking is used. |
| **Transaction Management** | The test relies on the framework’s default transaction handling (e.g., Spring’s `@Transactional` or CDI transaction). |
| **Test Isolation** | No cleanup logic is present; repeated runs may leave stale data. |
| **Hard‑coded Values** | Store postal code, product SKU, weight brackets, etc., are hard‑coded for simplicity. |
| **Ignored Test** | The method is annotated with both `@Ignore` and the class is ignored, so it won’t run unless the annotations are removed. |

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `testGetCustomShippingQuotesByWeight()` | Main integration test that validates weight‑based shipping quote calculation. | None (uses injected services) | `void` | Creates product, availability, pricing, shipping config, customer, and finally calls `shippingService.getShippingQuote`. Persists all entities to the database. |
| *Implicit* constructor | Provides an empty constructor via the framework. | – | – | – |

The test method is the only functional entry point; all other logic is delegated to domain services.

---

## 4. Dependencies

| Category | Dependency | Notes |
|----------|------------|-------|
| **Core Framework** | Spring (for `Assert`) | Minimal use – mainly for assertion; could be replaced by JUnit assertions. |
| **Injection** | CDI / Spring (`@Inject`) | Relies on container for wiring services. |
| **Domain Model** | `com.salesmanager.core.*` packages | Contains business services (`ShippingService`, `ProductService`, etc.) and domain entities (`Product`, `ShippingConfiguration`, etc.). |
| **Testing** | JUnit (`@Ignore`, `@Test`) | No external mocking libraries (e.g., Mockito) are used. |
| **Persistence** | Underlying JPA/Hibernate (implied) | Not explicit but inferred from service layer. |

All dependencies are either **third‑party** (Spring, JUnit) or **internal** to the SalesManager project.

---

## 5. Additional Notes & Recommendations

### Strengths

* **End‑to‑end Flow** – The test covers a full request path: from product creation to quote retrieval.  
* **Clear Separation** – Business logic is delegated to services, keeping the test readable.  
* **Explicit Configuration** – Shipping modules, regions, and weight brackets are fully defined in code, making the scenario transparent.

### Potential Issues & Edge Cases

1. **Test Isolation & Repeatability**  
   * No cleanup (`delete`/`rollback`) is performed. Running the test multiple times will create duplicate entries (e.g., product with SKU “TESTSKU”).  
   * Recommendation: Wrap the test in a transaction with `@Transactional` and `@Rollback` (if using Spring), or manually delete created entities in a `finally` block.

2. **Hard‑coded IDs / Strings**  
   * Hard‑coded SKUs, postal codes, and region codes make the test brittle if the underlying catalog changes.  
   * Recommendation: Use constants or generate random values where appropriate.

3. **Ignored Test**  
   * The test method and class are both annotated with `@Ignore`, meaning it never runs.  
   * Recommendation: Remove the `@Ignore` annotations once the test is ready for execution.

4. **Lack of Assertions**  
   * Only a non‑null check is performed on the shipping quote.  
   * Recommendation: Verify the quote’s price, shipping method, and other relevant fields to ensure the weight‑based calculation is correct.

5. **Mixing Domain Logic and Test Setup**  
   * The test is responsible for persisting a large amount of domain data, which can obscure the intent of the test.  
   * Recommendation: Extract setup logic into helper methods or a fixture factory to keep the test focused on behavior.

6. **Performance Concerns**  
   * Persisting many objects in a single test can be slow, especially if a real database is used.  
   * Recommendation: Use an in‑memory database (H2) for tests, or mock services where possible.

### Future Enhancements

| Enhancement | Benefit |
|-------------|---------|
| **Parameterized Tests** | Validate multiple weight brackets and destination countries in a single test method. |
| **Mocking Service Layers** | Reduce test execution time and isolate the shipping logic from persistence. |
| **Automated Cleanup** | Ensure a clean state between test runs. |
| **Structured Assertions** | Verify quote details (price, delivery time, method) to strengthen test coverage. |
| **Configuration File** | Load shipping configurations from a YAML/JSON file instead of hard‑coding in Java. |

---

### Verdict

The test demonstrates a solid understanding of the domain and successfully orchestrates a complex shipping quote scenario. However, its practical utility is limited by the lack of cleanup, minimal assertions, and the current `@Ignore` annotations. By addressing the points above, the test can evolve into a robust regression check that guarantees the correctness of weight‑based shipping calculations across the system.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shipping;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Set;

import javax.inject.Inject;

import org.junit.Ignore;
import org.springframework.util.Assert;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.business.services.shipping.ShippingService;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.description.ProductDescription;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPriceDescription;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.common.Billing;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.customer.CustomerGender;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.core.model.shipping.ShippingBasisType;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingPackageType;
import com.salesmanager.core.model.shipping.ShippingProduct;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.shipping.ShippingType;
import com.salesmanager.core.model.system.Environment;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.modules.integration.shipping.model.CustomShippingQuoteWeightItem;
import com.salesmanager.core.modules.integration.shipping.model.CustomShippingQuotesConfiguration;
import com.salesmanager.core.modules.integration.shipping.model.CustomShippingQuotesRegion;

@Ignore
public class ShippingQuoteByWeightTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	
	private static final Date date = new Date(System.currentTimeMillis());
	
	@Inject
	private ShippingService shippingService;
	
	@Inject
	private LanguageService languageService;
	

	
	
	@Ignore
	//@Test
	public void testGetCustomShippingQuotesByWeight() throws ServiceException {

	    Language en = languageService.getByCode("en");
	    Country country = countryService.getByCode("CA");
	    Zone zone = zoneService.getByCode("QC");

	    MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
	    ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);
	    
	    //set valid store postal code
	    store.setStorepostalcode("J4B-9J9");

	    Product product = new Product();
	    product.setProductHeight(new BigDecimal(4));
	    product.setProductLength(new BigDecimal(3));
	    product.setProductWidth(new BigDecimal(5));
	    product.setProductWeight(new BigDecimal(8));
	    product.setSku("TESTSKU");
	    product.setType(generalType);
	    product.setMerchantStore(store);

	    // Product description
	    ProductDescription description = new ProductDescription();
	    description.setName("Product 1");
	    description.setLanguage(en);
	    description.setProduct(product);

	    product.getDescriptions().add(description);
	    
	    productService.saveProduct(product);
	    //productService.saveOrUpdate(product);
	    

	    // Availability
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(new Date());
	    availability.setProductQuantity(100);
	    availability.setRegion("*");
	    availability.setProduct(product);// associate with product
	    
	    product.getAvailabilities().add(availability);

	    productAvailabilityService.create(availability);

	    ProductPrice dprice = new ProductPrice();
	    dprice.setDefaultPrice(true);
	    dprice.setProductPriceAmount(new BigDecimal(29.99));
	    dprice.setProductAvailability(availability);

	    ProductPriceDescription dpd = new ProductPriceDescription();
	    dpd.setName("Base price");
	    dpd.setProductPrice(dprice);
	    dpd.setLanguage(en);

	    dprice.getDescriptions().add(dpd);
	    availability.getPrices().add(dprice);
	    
	    productPriceService.create(dprice);
	    
	    //get product
	    product = productService.getBySku("TESTSKU", store, en);


	    
	    
	    //check the product
	    Set<ProductAvailability> avails = product.getAvailabilities();
	    for(ProductAvailability as : avails) {
		    Set<ProductPrice> availabilityPrices = as.getPrices();
		    for(ProductPrice ps : availabilityPrices) {
		    	System.out.println(ps.getProductPriceAmount().toString());
		    }
	    }
	    
	    //check availability
	    Set<ProductPrice> availabilityPrices = availability.getPrices();
	    for(ProductPrice ps : availabilityPrices) {
	    	System.out.println(ps.getProductPriceAmount().toString());
	    }
	    

	    
	    //configure shipping
	    ShippingConfiguration shippingConfiguration = new ShippingConfiguration();
	    shippingConfiguration.setShippingBasisType(ShippingBasisType.SHIPPING);//based on shipping or billing address
	    shippingConfiguration.setShippingType(ShippingType.INTERNATIONAL);
	    shippingConfiguration.setShippingPackageType(ShippingPackageType.ITEM);//individual item pricing or box packaging (see unit test above)
	    //only if package type is package
	    shippingConfiguration.setBoxHeight(5);
	    shippingConfiguration.setBoxLength(5);
	    shippingConfiguration.setBoxWidth(5);
	    shippingConfiguration.setBoxWeight(1);
	    shippingConfiguration.setMaxWeight(10);
	    
	    List<String> supportedCountries = new ArrayList<String>();
	    supportedCountries.add("CA");
	    supportedCountries.add("US");
	    supportedCountries.add("UK");
	    supportedCountries.add("FR");
	    
	    shippingService.setSupportedCountries(store, supportedCountries);
	    

	    CustomShippingQuotesConfiguration customConfiguration = new CustomShippingQuotesConfiguration();
		customConfiguration.setModuleCode("weightBased");
		customConfiguration.setActive(true);
		
		CustomShippingQuotesRegion northRegion = new CustomShippingQuotesRegion();
		northRegion.setCustomRegionName("NORTH");
		
		List<String> countries = new ArrayList<String>();
		countries.add("CA");
		countries.add("US");
		
		northRegion.setCountries(countries);
		
		CustomShippingQuoteWeightItem caQuote4 = new CustomShippingQuoteWeightItem();
		caQuote4.setMaximumWeight(4);
		caQuote4.setPrice(new BigDecimal(20));
		CustomShippingQuoteWeightItem caQuote10 = new CustomShippingQuoteWeightItem();
		caQuote10.setMaximumWeight(10);
		caQuote10.setPrice(new BigDecimal(50));
		CustomShippingQuoteWeightItem caQuote100 = new CustomShippingQuoteWeightItem();
		caQuote100.setMaximumWeight(100);
		caQuote100.setPrice(new BigDecimal(120));
		List<CustomShippingQuoteWeightItem> quotes = new ArrayList<CustomShippingQuoteWeightItem>();
		quotes.add(caQuote4);
		quotes.add(caQuote10);
		quotes.add(caQuote100);
		
		northRegion.setQuoteItems(quotes);
		
		customConfiguration.getRegions().add(northRegion);
	    
	    
	    //create an integration configuration - USPS
	    
	    IntegrationConfiguration configuration = new IntegrationConfiguration();
	    configuration.setActive(true);
	    configuration.setEnvironment(Environment.TEST.name());
	    configuration.setModuleCode("weightBased");
	    
	    //configure module



	    shippingService.saveShippingConfiguration(shippingConfiguration, store);
	    shippingService.saveShippingQuoteModuleConfiguration(configuration, store);//create the basic configuration
	    shippingService.saveCustomShippingConfiguration("weightBased", customConfiguration, store);//and the custom configuration
	    
	    //now create ShippingProduct
	    ShippingProduct shippingProduct1 = new ShippingProduct(product);
	    FinalPrice price = pricingService.calculateProductPrice(product);
	    shippingProduct1.setFinalPrice(price);
	    
	    List<ShippingProduct> shippingProducts = new ArrayList<ShippingProduct>();
	    shippingProducts.add(shippingProduct1);
	    
		Customer customer = new Customer();
		customer.setMerchantStore(store);
		customer.setEmailAddress("test@test.com");
		customer.setGender(CustomerGender.M);
		customer.setDefaultLanguage(en);

		customer.setAnonymous(true);
		customer.setCompany("ifactory");
		customer.setDateOfBirth(new Date());
		customer.setNick("My nick");
		customer.setPassword("123456");

		
	    Delivery delivery = new Delivery();
	    delivery.setAddress("Shipping address");
	    delivery.setCity("Boucherville");
	    delivery.setCountry(country);
	    delivery.setZone(zone);
	    delivery.setPostalCode("J5C-6J4");
	    
	    //overwrite delivery to US
/*	    delivery.setPostalCode("90002");
	    delivery.setCountry(us);
	    Zone california = zoneService.getByCode("CA");
	    delivery.setZone(california);*/
	    
	    
	    Billing billing = new Billing();
	    billing.setAddress("Billing address");
	    billing.setCountry(country);
	    billing.setZone(zone);
	    billing.setPostalCode("J4B-8J9");
	    billing.setFirstName("Carl");
	    billing.setLastName("Samson");
	    
	    customer.setBilling(billing);
	    customer.setDelivery(delivery);
		
		customerService.create(customer);
		
		Long dummyCartId = 0L;//for correlation
	    
	    ShippingQuote shippingQuote = shippingService.getShippingQuote(dummyCartId, store, delivery, shippingProducts, en);

	    Assert.notNull(shippingQuote);
	    
	}



}


```
