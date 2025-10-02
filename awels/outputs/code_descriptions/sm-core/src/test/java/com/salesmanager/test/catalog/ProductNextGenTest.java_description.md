# ProductNextGenTest.java

## Review

## 1. Summary  
The test class **`ProductNextGenTest`** demonstrates the end‑to‑end creation of a variable product (a pair of shoes with size variants) using the SalesManager core APIs.  

* **Purpose** – Verify that a product with categories, manufacturer, option sets, attributes, prices and inventory can be persisted and retrieved correctly.  
* **Key components**  
  * **Domain entities** – `Product`, `Category`, `Manufacturer`, `ProductOption`, `ProductOptionValue`, `ProductAttribute`, `ProductAvailability`, `ProductPrice` etc.  
  * **Service layer** – `categoryService`, `manufacturerService`, `productService`, `productOptionService`, `productOptionValueService`, `productOptionSetService`, etc.  
  * **Helper methods** – `createOptions`, `createOptionsSet`, `createInventory` – encapsulate complex entity creation logic.  
* **Design patterns** – The code largely follows the **Domain‑Driven Design** (DDD) model used by SalesManager: entities are rich objects with collections of value objects, and services act as application facades.  
* **Frameworks** – Spring (for dependency injection), JUnit + Hamcrest for assertions, and the SalesManager core libraries.

---

## 2. Detailed Description  

### Execution Flow  
1. **Setup** – Retrieve the default language and store, and the general product type.  
2. **Category & Manufacturer** – Create a `shoes` category and a `brown` manufacturer, persisting them via the respective services.  
3. **Base Product** – Instantiate `Product` with basic attributes (height, length, width, SKU, etc.), attach descriptions, add to category, and flag as available.  
4. **Options & Values** –  
   * `createOptions` builds the *size* option and three values (9, 9.5, 10) and persists them.  
   * `createOptionsSet` registers a reusable set `majorSizes` that includes 9 and 10.  
5. **Product Attributes** – Three `ProductAttribute` instances are created, linking the product to each size value.  
6. **Inventory** – `createInventory` builds a `ProductAvailability` (price, quantity, region) and associates it with the product.  
7. **Persist** – `productService.saveProduct` persists the entire object graph.  
8. **Verification** –  
   * Retrieve the product by ID and assert it is not null.  
   * List products by category and by option value, asserting non‑empty results.  

### Assumptions & Constraints  
* The test assumes a **single store** (the default store) and a **single language** (“en”).  
* It presumes that the underlying persistence layer (JPA/Hibernate) is correctly configured and that the test context loads all SalesManager services.  
* Inventory is simplified: only a single `ProductAvailability` with a default price is created; variant‑level pricing/quantity is commented out.  
* No transaction rollback is used; the test relies on the test framework’s default behavior (usually rollback after each test).  

### Architecture  
The code follows a **service‑centric architecture**: each domain concept has a dedicated service (`categoryService`, `manufacturerService`, etc.). The test orchestrates these services to build a complete product hierarchy, demonstrating how the domain model interacts with the persistence layer.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `testCreateProduct()` | Main integration test for product creation | None (uses injected services) | `void` (assertions only) | Persists entities; reads from DB for assertions |
| `createOptionsSet(MerchantStore)` | Creates a reusable `ProductOptionSet` containing size options | `store` | `void` | Persists `possibleSizes` |
| `createInventory(MerchantStore, int, BigDecimal)` | Builds a `ProductAvailability` with default price | `store`, `quantity`, `price` | `ProductAvailability` | Sets dates, region, price description |
| `createOptions(Product)` | Creates size option and its values (9, 9.5, 10) | `product` (used for merchant store) | `void` | Persists option and values |

**Utility Notes**  
* The helper methods encapsulate repetitive logic, making the test easier to read.  
* All entity creations are performed in a *builder‑like* style: set fields, add to collections, then persist.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.junit` | Third‑party | JUnit 4 (annotation `@Test`) |
| `org.hamcrest` | Third‑party | Hamcrest matchers for assertions |
| `org.springframework.data.domain.Page` | Spring Data | Pagination support |
| `com.salesmanager.core.*` | Third‑party | Core SalesManager domain, services, exceptions |
| `java.math.BigDecimal`, `java.sql.Date`, `java.util.*` | JDK | Standard library |

*Platform‑specific*: None beyond standard Java and Spring; the test relies on the SalesManager test base (`AbstractSalesManagerCoreTestCase`) for context initialization.

---

## 5. Additional Notes & Recommendations  

### Strengths  
* **Comprehensive coverage** – The test exercises a large part of the product domain: categories, manufacturers, options, attributes, pricing, and inventory.  
* **Clear separation** – Helper methods isolate concerns, improving readability.  
* **Use of constants** – `date` and language/store look‑ups keep the test concise.  

### Areas for Improvement  

1. **Magic strings & numbers**  
   * Codes like `"shoes"`, `"brown"`, `"nine"` are hard‑coded. Introduce constants or enums to avoid typos.  
   * Quantities (40, 30, 30) are commented out; if variant inventory is needed, make them parameters or configuration.

2. **Exception handling**  
   * The test declares `throws Exception`. Prefer more specific exceptions (`ServiceException`) to aid debugging.  

3. **Transactional context**  
   * Rely on Spring’s `@Transactional` with rollback to avoid side‑effects on other tests.  

4. **Assertions**  
   * Add deeper checks: verify product descriptions, attribute count, price correctness, availability quantity.  
   * Use Hamcrest matchers for nested collections (e.g., `hasSize(1)` on `product.getCategories()`).

5. **Code duplication**  
   * The creation of `ProductOptionDescription`, `ProductOptionValueDescription`, etc., repeats similar patterns. Consider a small builder helper.

6. **Logging**  
   * Add optional logging (e.g., `log.debug`) for entity IDs after persistence, helpful in failing tests.

7. **Resource cleanup**  
   * Although the test framework rolls back, explicit cleanup (deleting the created product) can make the test idempotent if run outside the framework.

8. **Commented code**  
   * Remove or document why the `ProductVariant` sections are commented out; if they’re no longer needed, delete them.

### Future Enhancements  

* **Variant Pricing** – Uncomment and implement variant‑level pricing logic.  
* **Multiple Languages** – Add a second language to test multilingual support.  
* **Multiple Stores** – Extend the test to a multi‑store scenario.  
* **Performance** – Measure the time to create and retrieve many products; ensure scalability.  

---

**Overall Verdict** – The test demonstrates solid usage of the SalesManager API and is a good reference for end‑to‑end product creation. Minor refactoring for maintainability and richer assertions would elevate its robustness and readability.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.catalog;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.empty;
import static org.hamcrest.Matchers.not;
import static org.junit.Assert.assertNotNull;

import java.math.BigDecimal;
import java.sql.Date;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import org.junit.Test;
import org.springframework.data.domain.Page;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.category.CategoryDescription;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.ProductCriteria;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.attribute.ProductOption;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionDescription;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionSet;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionType;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionValue;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionValueDescription;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.description.ProductDescription;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.manufacturer.ManufacturerDescription;
import com.salesmanager.core.model.catalog.product.price.ProductPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPriceDescription;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;


public class ProductNextGenTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	
	private static final Date date = new Date(System.currentTimeMillis());
	
	private ProductOptionValue nine = null;
	private ProductOptionValue nineHalf = null;
	private ProductOptionValue ten = null;
	private ProductOption size = null;
	
	private ProductOptionSet possibleSizes;
	
	

	/**
	 * This method creates single product with variants using multiple catalog APIs
	 * @throws ServiceException
	 */
	@Test
	public void testCreateProduct() throws Exception {

	    Language en = languageService.getByCode("en");

	    MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
	    ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);

	    /**
	     * Category
	     */
	    Category shoes = new Category();
	    shoes.setMerchantStore(store);
	    shoes.setCode("shoes");

	    CategoryDescription shoesDescription = new CategoryDescription();
	    shoesDescription.setName("Shoes");
	    shoesDescription.setCategory(shoes);
	    shoesDescription.setLanguage(en);

	    Set<CategoryDescription> descriptions = new HashSet<CategoryDescription>();
	    descriptions.add(shoesDescription);

	    shoes.setDescriptions(descriptions);

	    categoryService.create(shoes);
	    //
	    
	    /**
	     * Manufacturer / Brand
	     */

	    Manufacturer brown = new Manufacturer();
	    brown.setMerchantStore(store);
	    brown.setCode("brown");

	    ManufacturerDescription brownd = new ManufacturerDescription();
	    brownd.setLanguage(en);
	    brownd.setName("Brown's");
	    brownd.setManufacturer(brown);
	    brown.getDescriptions().add(brownd);

	    manufacturerService.create(brown);
	    //

	    
	    // PRODUCT
	    
	    // -- non variable informations

	    Product summerShoes = new Product();
	    summerShoes.setProductHeight(new BigDecimal(3));
	    summerShoes.setProductLength(new BigDecimal(9));//average
	    summerShoes.setProductWidth(new BigDecimal(4));
	    summerShoes.setSku("BR12345");
	    summerShoes.setManufacturer(brown);
	    summerShoes.setType(generalType);
	    summerShoes.setMerchantStore(store);
	    
	    //is available
	    summerShoes.setAvailable(true);

	    // Product description
	    ProductDescription description = new ProductDescription();
	    description.setName("Summer shoes");
	    description.setLanguage(en);
	    description.setProduct(summerShoes);

	    summerShoes.getDescriptions().add(description);

	    //add product to shoes category
	    summerShoes.getCategories().add(shoes);
	    
	    // -- end non variable informations
	    
	    // --- add attributes to the product (size)
	    createOptions(summerShoes);
	    
	    // --- options set
	    /**
	     * Option Set facilitates product attributes creation for redundant product creation
	     * it offers a list of possible options and options values administrator can create in order
	     * to easily create attributes 
	     */
	    createOptionsSet(store);
	    
	    // --- create attributes (available options)
	    /**
	     * Add options to product
	     * Those are attributes
	     */
        
        ProductAttribute size_nine = new ProductAttribute();
        size_nine.setProduct(summerShoes);
        size_nine.setProductOption(size);
        size_nine.setAttributeDefault(true);
        size_nine.setProductAttributePrice(new BigDecimal(0));//no price variation
        size_nine.setProductAttributeWeight(new BigDecimal(0));//no weight variation
        size_nine.setProductOptionValue(nine);
        
        summerShoes.getAttributes().add(size_nine);
        

	    ProductAttribute size_nine_half = new ProductAttribute();
	    size_nine_half.setProduct(summerShoes);
	    size_nine_half.setProductOption(size);
	    size_nine_half.setProductAttributePrice(new BigDecimal(0));//no price variation
	    size_nine_half.setProductAttributeWeight(new BigDecimal(0));//weight variation
	    size_nine_half.setProductOptionValue(nineHalf);
	    
	    summerShoes.getAttributes().add(size_nine_half);


	    ProductAttribute size_ten = new ProductAttribute();
	    size_ten.setProduct(summerShoes);
	    size_ten.setProductOption(size);
	    size_ten.setProductAttributePrice(new BigDecimal(0));//no price variation
	    size_ten.setProductAttributeWeight(new BigDecimal(0));//weight variation
	    size_ten.setProductOptionValue(ten);
	    
	    summerShoes.getAttributes().add(size_ten);	    
	    
	    
	    // ---- variable informations - inventory - variants - prices
	    ProductAvailability availability = createInventory(store, 100, new BigDecimal("99.99"));
	    summerShoes.getAvailabilities().add(availability);
	    // ---- add available sizes
	    
	    //DEFAULT (total quantity of 100 distributed)
	    
	    //TODO use pre 3.0 variation
	    
	    //40 of 9
/*	    ProductVariant size_nine_DEFAULT = new ProductVariant();
	    size_nine_DEFAULT.setAttribute(size_nine);
	    size_nine_DEFAULT.setProductQuantity(40);
	    size_nine_DEFAULT.setProductAvailability(availability);*/
	    
	    //availability.getVariants().add(size_nine_DEFAULT);
	    
	    //30 of 9.5
/*	    ProductVariant size_nine_half_DEFAULT = new ProductVariant();
	    size_nine_half_DEFAULT.setAttribute(size_nine_half);
	    size_nine_half_DEFAULT.setProductQuantity(30);
	    size_nine_half_DEFAULT.setProductAvailability(availability);*/
	    
	    //availability.getVariants().add(size_nine_half_DEFAULT);
	    
	    //30 of ten
/*	    ProductVariant size_ten_DEFAULT = new ProductVariant();
	    size_ten_DEFAULT.setAttribute(size_nine_half);
	    size_ten_DEFAULT.setProductQuantity(30);
	    size_ten_DEFAULT.setProductAvailability(availability);*/
	    
	    //availability.getVariants().add(size_ten_DEFAULT);
	    
	    //inventory for store DEFAULT and product summerShoes
	    availability.setProduct(summerShoes);
	    availability.setMerchantStore(store);

	    
	    /**
	     * Create product
	     */
	    productService.saveProduct(summerShoes);

	    //ObjectMapper mapper = new ObjectMapper();
	    //Converting the Object to JSONString
	    //String jsonString = mapper.writeValueAsString(summerShoes);
	    //System.out.println(jsonString);
	    
	    Product p = productService.getById(summerShoes.getId());
	    assertNotNull(p);
	    
	    //List<ProductAvailability> avs = p.getAvailabilities().stream().filter(a -> !a.getVariants().isEmpty()).collect(Collectors.toList());
	    //assertThat(avs, not(empty()));
	    
	    //test product list service
	    
	    //list products per category
	    //list 5 items
	    ProductCriteria productCriteria = new ProductCriteria();
	    productCriteria.setCategoryIds(Stream.of(shoes.getId())
	    	      .collect(Collectors.toList()));
	    
	    Page<Product> listByCategory = productService.listByStore(store, en, productCriteria, 0, 5);
	    List<Product> productsByCategory = listByCategory.getContent();
	    assertThat(productsByCategory,  not(empty()));
	    
	    //list products per color attribute
	    Page<Product> listByOptionValue = productService.listByStore(store, en, productCriteria, 0, 5);
	    productCriteria = new ProductCriteria();
	    productCriteria.setOptionValueIds(Stream.of(nineHalf.getId())
	    	      .collect(Collectors.toList()));
	    List<Product> productsByOption = listByOptionValue.getContent();
	    assertThat(productsByOption,  not(empty()));
	    
	    

	}
	
	private void createOptionsSet(MerchantStore store) throws Exception {
		
		//add a set of option / values for major sizes
		possibleSizes = new ProductOptionSet();
		possibleSizes.setCode("majorSizes");
		possibleSizes.setStore(store);
		possibleSizes.setOption(size);
		
		
		List<ProductOptionValue> values = new ArrayList<ProductOptionValue>();
		values.add(nine);
		values.add(ten);
		
		possibleSizes.setValues(values);
		

		productOptionSetService.create(possibleSizes);
		
	}
	
	/**
	 * Manage product inventory
	 * @param product
	 * @throws Exception
	 */
	private ProductAvailability createInventory(MerchantStore store, int quantity, BigDecimal price) throws Exception {
		
		//add inventory
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(date);
	    availability.setProductQuantity(quantity);
	    availability.setRegion("*");
	    availability.setAvailable(true);
	    
	    Language en = languageService.getByCode("en");

	    ProductPrice dprice = new ProductPrice();
	    dprice.setDefaultPrice(true);
	    dprice.setProductPriceAmount(price);
	    dprice.setProductAvailability(availability);

	    ProductPriceDescription dpd = new ProductPriceDescription();
	    dpd.setName("Base price");
	    dpd.setProductPrice(dprice);
	    dpd.setLanguage(en);

	    dprice.getDescriptions().add(dpd);
	    availability.getPrices().add(dprice);
	    
	    return availability;
		
		
		
	}

	
	/**
	 * Add possible choices
	 * @param product
	 * @throws Exception
	 */
	private void createOptions(Product product) throws Exception {
		
		
		/**
		 * An attribute can be created dynamicaly but the attached Option and Option value need to exist
		 */
		
		MerchantStore store = product.getMerchantStore();
		Language en = languageService.getByCode("en");
		

	     /**
         * Create size option
         */
        size = new ProductOption();
        size.setMerchantStore(store);
        size.setCode("SHOESIZE");
        size.setProductOptionType(ProductOptionType.Radio.name());
        
        ProductOptionDescription sizeDescription = new ProductOptionDescription();
        sizeDescription.setLanguage(en);
        sizeDescription.setName("Size");
        sizeDescription.setDescription("Show size");
        sizeDescription.setProductOption(size);
        
        size.getDescriptions().add(sizeDescription);
        
        //create option
        productOptionService.saveOrUpdate(size);
	    
        
        /**
         * Create size values (9, 9.5, 10)
         */
	    
	    //option value 9
	    nine = new ProductOptionValue();
	    nine.setMerchantStore(store);
	    nine.setCode("nine");
	    
	    ProductOptionValueDescription nineDescription = new ProductOptionValueDescription();
	    nineDescription.setLanguage(en);
	    nineDescription.setName("9");
	    nineDescription.setDescription("Size 9");
	    nineDescription.setProductOptionValue(nine);
	    
	    nine.getDescriptions().add(nineDescription);
	    
	    //create an option value
	    productOptionValueService.saveOrUpdate(nine);
	    
	    
	    //option value 9.5
	    nineHalf = new ProductOptionValue();
	    nineHalf.setMerchantStore(store);
	    nineHalf.setCode("nineHalf");
	    
	    ProductOptionValueDescription nineHalfDescription = new ProductOptionValueDescription();
	    nineHalfDescription.setLanguage(en);
	    nineHalfDescription.setName("9.5");
	    nineHalfDescription.setDescription("Size 9.5");
	    nineHalfDescription.setProductOptionValue(nineHalf);
	    
	    nineHalf.getDescriptions().add(nineHalfDescription);
	    
	    //create an option value
	    productOptionValueService.saveOrUpdate(nineHalf);
	    
	    
	    //option value 10
	    ten = new ProductOptionValue();
	    ten.setMerchantStore(store);
	    ten.setCode("ten");
	    
	    ProductOptionValueDescription tenDescription = new ProductOptionValueDescription();
	    tenDescription.setLanguage(en);
	    tenDescription.setName("10");
	    tenDescription.setDescription("Size 10");
	    tenDescription.setProductOptionValue(ten);
	    
	    ten.getDescriptions().add(tenDescription);
	    
	    //create an option value
	    productOptionValueService.saveOrUpdate(ten);
	    
	    
	    // end options / options values


	}
	


	
	
	
	




}


```
