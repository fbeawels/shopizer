# ShoppingCartTest.java

## Review

## 1. Summary  

**Purpose** – The `ShoppingCartTest` class is a JUnit 4 integration test that exercises the full life‑cycle of a persistent shopping cart in the *SalesManager* e‑commerce platform.  
- It creates the required catalog artefacts (store, language, category, manufacturer, product options & values, a product with pricing, availability and attributes).  
- It then builds a `ShoppingCart`, adds a `ShoppingCartItem` with a selected option value, persists the cart, retrieves it, verifies it exists, deletes it, and finally asserts that it has been removed.  
- The test also cleans up the created category (and implicitly all dependent data via cascade).

**Key components**  
| Component | Role |
|-----------|------|
| `MerchantStore`, `Language` | Context for catalog items. |
| `Category`, `Manufacturer`, `ProductOption`, `ProductOptionValue` | Catalog definition objects. |
| `Product`, `ProductDescription`, `ProductAvailability`, `ProductPrice` | The product being sold. |
| `ProductAttribute` | Option/value combinations that alter price/weight. |
| `ShoppingCart`, `ShoppingCartItem`, `ShoppingCartAttributeItem` | The shopping cart entity and its line items. |
| `pricingService` | Calculates the final price for the product (including attribute variations). |
| `shoppingCartService` | Persist/retrieve/delete cart. |

The test leverages a number of *SalesManager* services (`merchantService`, `languageService`, `categoryService`, etc.) that are injected from the base test case `AbstractSalesManagerCoreTestCase`.

---

## 2. Detailed Description  

### Execution Flow  

1. **Context acquisition** – The test fetches the default `MerchantStore` and English `Language`.  
2. **Catalog creation** –  
   - A `Category` (“Shirts”) is created with a description.  
   - A `Manufacturer` (“Addidas”) is created with a description.  
   - A `ProductOption` (“Color”) and two `ProductOptionValue`s (“White”, “Black”) are defined.  
   - A complex `Product` is assembled: size, SKU, manufacturer, type, descriptions, category, availability, pricing, and two `ProductAttribute`s (default & black).  
   - All entities are persisted through their respective services.  
3. **Shopping cart assembly** –  
   - A new `ShoppingCart` is created with a random UUID as its code.  
   - A `ShoppingCartItem` references the product and sets quantity = 1.  
   - The `pricingService` calculates the final price; the item price is set accordingly.  
   - The user’s choice of the “Black” option is represented by a `ShoppingCartAttributeItem` attached to the line item.  
   - The cart is persisted via `shoppingCartService.create()`.  
4. **Validation** –  
   - The cart is retrieved by its code; a non‑null assertion confirms persistence.  
   - The cart is deleted; a subsequent retrieval should return `null`.  
5. **Cleanup** – The test removes the created `Category` (cascades to product, etc.) to keep the database tidy for other tests.

### Design Choices  

* **Explicit entity construction** – The test constructs each entity manually, setting every field and relationship.  
* **No transaction management** – The test relies on the underlying framework to commit each service call; there is no explicit rollback.  
* **Single test method** – All catalog and cart operations live in one `@Test` method.  
* **Hard‑coded strings** – Store code (`MerchantStore.DEFAULT_STORE`), language code (`"en"`), and product data are hard‑coded, making the test brittle if the baseline data changes.  

### Assumptions & Constraints  

* A store with code `MerchantStore.DEFAULT_STORE` already exists in the test database.  
* The default language “en” is present.  
* The services used (`merchantService`, `languageService`, …) are correctly wired by the base test class.  
* Entity relationships and cascading are properly configured so that a `categoryService.delete()` will cascade to the product, manufacturer, options, etc.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `createShoppingCart()` (annotated with `@Test`) | Executes the entire cart life‑cycle test. | None | None (assertions performed within) | Persists catalog data, creates/deletes a cart, deletes the category. |

*Helper objects* (no separate methods but significant objects):  
* `ProductOption`, `ProductOptionValue`, `ProductAttribute`, `ShoppingCartAttributeItem` – all represent option/value associations in the cart.  
* `FinalPrice` – returned by `pricingService.calculateProductPrice()` and used to set the line‑item price.

---

## 4. Dependencies  

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `org.junit.Test`, `org.junit.Assert` | Third‑party (JUnit 4) | Test framework. |
| `com.salesmanager.core.model.*` | Internal | Domain entities (category, product, cart, etc.). |
| `com.salesmanager.core.model.catalog.*` | Internal | Catalog‑specific entities. |
| `com.salesmanager.core.model.shoppingcart.*` | Internal | Cart‑specific entities. |
| `com.salesmanager.core.*` services (`merchantService`, `categoryService`, etc.) | Internal | Business‑logic services injected via the base test case. |
| `java.math.BigDecimal`, `java.util.*` | Standard | Utility types. |

No external libraries beyond JUnit and the SalesManager core modules are used.

---

## 5. Additional Notes  

### Strengths  

* **End‑to‑end coverage** – The test exercises the full path from catalog creation to cart persistence, ensuring that all related services and mappings work together.  
* **Self‑contained data** – All entities are created within the test, making it repeatable in isolation.  
* **Clear assertions** – The test confirms both the existence and subsequent deletion of the cart.

### Potential Issues & Edge Cases  

1. **Tight coupling to default data** – The test assumes a default store and language. If these are removed or renamed, the test will fail even though the cart logic is fine.  
2. **Cascading assumptions** – The cleanup only deletes the `Category`. If cascading is not properly configured, orphaned entities may remain, potentially affecting other tests.  
3. **Lack of transactional rollback** – The test does not wrap operations in a transaction that rolls back on failure. A failure after the cart is created would leave residual data.  
4. **Hard‑coded values** – SKU, option codes, etc., are hard‑coded; changes to business rules could render the test obsolete.  
5. **No verification of pricing logic** – The test asserts the cart exists but does not verify that the calculated price matches the expected value (29.99 + 5 for black).  
6. **Single method, no setup/teardown** – Splitting the test into `@Before` (catalog creation), `@After` (cleanup) would improve readability and isolation.

### Recommendations for Improvement  

| Area | Suggested Change |
|------|------------------|
| **Test Structure** | Split catalog creation into a `@Before` method, cart operations into the `@Test`, and cleanup into `@After`. |
| **Transactional Control** | Annotate the test (or the test class) with `@Transactional` and `@Rollback` to automatically revert all changes on test completion. |
| **Assertions** | Add assertions for the final price, quantity, selected option, and product reference to verify business logic beyond persistence. |
| **Reusable Factories** | Extract helper methods (e.g., `createCategory`, `createManufacturer`, `createProductOption`) to reduce duplication and improve clarity. |
| **Configuration Flexibility** | Load store and language codes from a properties file or use the `merchantService` to create a temporary store for the test, avoiding reliance on existing data. |
| **Error Handling** | While JUnit will fail on exceptions, consider wrapping critical sections in try/catch to provide clearer diagnostics if persistence fails. |
| **Performance** | If the catalog data is large, consider using in‑memory H2 and resetting the schema after each test. |

---

### Bottom Line  

The `ShoppingCartTest` demonstrates a pragmatic, albeit verbose, integration test for the SalesManager shopping cart functionality. It validates that a complex product can be persisted, a cart can be created with option selections, and that persistence and deletion work as expected. Enhancing the test with transactional rollback, clearer structure, and richer assertions would improve maintainability, robustness, and confidence in the underlying pricing logic.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shoppingcart;

import java.math.BigDecimal;
import java.util.Date;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

import org.junit.Assert;
import org.junit.Test;

import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.category.CategoryDescription;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.attribute.ProductOption;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionDescription;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionType;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionValue;
import com.salesmanager.core.model.catalog.product.attribute.ProductOptionValueDescription;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.description.ProductDescription;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.manufacturer.ManufacturerDescription;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPriceDescription;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.shoppingcart.ShoppingCart;
import com.salesmanager.core.model.shoppingcart.ShoppingCartAttributeItem;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;



/**
 * Test 
 * 
 * - Add a product to persistent shopping cart
 * - Retrieve an item from the persistent shopping cart
 * - Rebuild a shopping cart item after the product definition has been modified
 * @author Carl Samson
 *
 */
public class ShoppingCartTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	


	@Test
	public void createShoppingCart() throws Exception {

        MerchantStore store = merchantService.getByCode( MerchantStore.DEFAULT_STORE );
        
		
	    Language en = languageService.getByCode("en");


	    /** CATALOG CREATION **/
	    
	    ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);

	    /**
	     * Create the category
	     */
	    Category shirts = new Category();
	    shirts.setMerchantStore(store);
	    shirts.setCode("shirts");

	    CategoryDescription shirtsEnglishDescription = new CategoryDescription();
	    shirtsEnglishDescription.setName("Shirts");
	    shirtsEnglishDescription.setCategory(shirts);
	    shirtsEnglishDescription.setLanguage(en);

	    Set<CategoryDescription> descriptions = new HashSet<CategoryDescription>();
	    descriptions.add(shirtsEnglishDescription);


	    shirts.setDescriptions(descriptions);
	    categoryService.create(shirts);
	    
	    
	    /**
	     * Create a manufacturer
	     */
	    Manufacturer addidas = new Manufacturer();
	    addidas.setMerchantStore(store);
	    addidas.setCode("addidas");

	    ManufacturerDescription addidasDesc = new ManufacturerDescription();
	    addidasDesc.setLanguage(en);
	    addidasDesc.setManufacturer(addidas);
	    addidasDesc.setName("Addidas");
	    addidas.getDescriptions().add(addidasDesc);

	    manufacturerService.create(addidas);
	    
	    /**
	     * Create an option
	     */
	    ProductOption option = new ProductOption();
	    option.setMerchantStore(store);
	    option.setCode("color");
	    option.setProductOptionType(ProductOptionType.Radio.name());
	    
	    ProductOptionDescription optionDescription = new ProductOptionDescription();
	    optionDescription.setLanguage(en);
	    optionDescription.setName("Color");
	    optionDescription.setDescription("Item color");
	    optionDescription.setProductOption(option);
	    
	    option.getDescriptions().add(optionDescription);
	    
	    productOptionService.saveOrUpdate(option);
	    
	    
	    /** first option value **/
	    ProductOptionValue white = new ProductOptionValue();
	    white.setMerchantStore(store);
	    white.setCode("white");
	    
	    ProductOptionValueDescription whiteDescription = new ProductOptionValueDescription();
	    whiteDescription.setLanguage(en);
	    whiteDescription.setName("White");
	    whiteDescription.setDescription("White color");
	    whiteDescription.setProductOptionValue(white);
	    
	    white.getDescriptions().add(whiteDescription);
	    
	    productOptionValueService.saveOrUpdate(white);
	    
	    
	    ProductOptionValue black = new ProductOptionValue();
	    black.setMerchantStore(store);
	    black.setCode("black");
	    
	    /** second option value **/
	    ProductOptionValueDescription blackDesc = new ProductOptionValueDescription();
	    blackDesc.setLanguage(en);
	    blackDesc.setName("Black");
	    blackDesc.setDescription("Black color");
	    blackDesc.setProductOptionValue(black);
	    
	    black.getDescriptions().add(blackDesc);

	    productOptionValueService.saveOrUpdate(black);
	    
	    
	    /**
	     * Create a complex product
	     */
	    Product product = new Product();
	    product.setProductHeight(new BigDecimal(4));
	    product.setProductLength(new BigDecimal(3));
	    product.setProductWidth(new BigDecimal(1));
	    product.setSku("XABC12");
	    product.setManufacturer(addidas);
	    product.setType(generalType);
	    product.setMerchantStore(store);

	    // Product description
	    ProductDescription description = new ProductDescription();
	    description.setName("Short sleeves shirt");
	    description.setLanguage(en);
	    description.setProduct(product);

	    product.getDescriptions().add(description);
	    product.getCategories().add(shirts);
	    
	    
	    //availability
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(new Date());
	    availability.setProductQuantity(100);
	    availability.setRegion("*");
	    availability.setProduct(product);// associate with product
	    
	    //price
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
	    product.getAvailabilities().add(availability);
	    
	    
	    //attributes
	    //white
	    ProductAttribute whiteAttribute = new ProductAttribute();
	    whiteAttribute.setProduct(product);
	    whiteAttribute.setProductOption(option);
	    whiteAttribute.setAttributeDefault(true);
	    whiteAttribute.setProductAttributePrice(new BigDecimal(0));//no price variation
	    whiteAttribute.setProductAttributeWeight(new BigDecimal(0));//no weight variation
	    whiteAttribute.setProductOption(option);
	    whiteAttribute.setProductOptionValue(white);
	    
	    product.getAttributes().add(whiteAttribute);
	    //black
	    ProductAttribute blackAttribute = new ProductAttribute();
	    blackAttribute.setProduct(product);
	    blackAttribute.setProductOption(option);
	    blackAttribute.setProductAttributePrice(new BigDecimal(5));//5 + dollars
	    blackAttribute.setProductAttributeWeight(new BigDecimal(0));//no weight variation
	    blackAttribute.setProductOption(option);
	    blackAttribute.setProductOptionValue(black);
	    
	    product.getAttributes().add(blackAttribute);

	    productService.saveProduct(product);
	    
	    /** Create Shopping cart **/
	    
	    ShoppingCart shoppingCart = new ShoppingCart();
	    shoppingCart.setMerchantStore(store);
	    
	    UUID cartCode = UUID.randomUUID();
	    shoppingCart.setShoppingCartCode(cartCode.toString());

	    ShoppingCartItem item = new ShoppingCartItem(shoppingCart,product);
	    item.setSku(product.getSku());
	    item.setShoppingCart(shoppingCart);
	    
	    FinalPrice price = pricingService.calculateProductPrice(product);

	    item.setItemPrice(price.getFinalPrice());
	    item.setQuantity(1);
	    
	    /** user selects black **/
	    ShoppingCartAttributeItem attributeItem = new ShoppingCartAttributeItem(item,blackAttribute);
	    item.getAttributes().add(attributeItem);
	    
	    shoppingCart.getLineItems().add(item);

	    
	    //create cart
	    shoppingCartService.create(shoppingCart);

	    
	    /** Retrieve cart **/

	    ShoppingCart retrievedCart = shoppingCartService.getByCode(cartCode.toString(), store);
	    
	    Assert.assertNotNull(retrievedCart);
	    
	    /** Delete cart **/
	    shoppingCartService.delete(retrievedCart);
	    
	    /** Check if cart has been deleted **/
	    retrievedCart = shoppingCartService.getByCode(cartCode.toString(), store);
	    
	    Assert.assertNull(retrievedCart);

		// Clean up for other tests
	    categoryService.delete(shirts);
	    
	}
	

}


```
