# ProductTest.java

## Review

## 1. Summary  

**Purpose**  
`ProductTest` is a JUnit test case that exercises the full lifecycle of a product in the *SalesManager* core domain:  

- Creation of categories, manufacturers, and a product (with descriptions, pricing, availability, images, attributes, and relationships).  
- Retrieval and counting of products by category lineage.  
- Update of pricing information.  
- Basic validation of image handling.  
- Cleanup of created data.

**Key Components**  

| Component | Role |
|-----------|------|
| `AbstractSalesManagerCoreTestCase` | Provides access to service beans (`productService`, `categoryService`, etc.) and sets up the Spring test context. |
| Domain entities (`Product`, `Category`, `Manufacturer`, …) | Represent catalog objects with bi‑directional relationships. |
| Service layers (`productService`, `categoryService`, …) | Persist, query, and delete entities. |
| `Language`, `MerchantStore` | Contextual data used for internationalisation and multi‑store support. |
| JUnit `@Test` | Drives the test flow. |

**Notable Design Patterns / Libraries**  

- **Service‑Repository pattern** – business logic lives in service beans, persistence in JPA repositories.  
- **Factory / Builder** – although not explicitly used, the test constructs entities in a “hand‑crafted” way.  
- **JUnit 4** – the test uses `@Test` without any parameterized or lifecycle annotations.  
- **Spring Test Context** – injection of services via the abstract base class.  
- **Java 8+** – uses `java.math.BigDecimal`, `java.util` collections, and `java.sql.Date`.

---

## 2. Detailed Description  

### 2.1 Execution Flow  

1. **Setup** – The test class extends `AbstractSalesManagerCoreTestCase`, so Spring injects all services.  
2. **`testCreateProduct`** –  
   - Loads English & French languages and the default store.  
   - Builds four categories (`book`, `music`, `novell`, `tech`, `fiction`) with translations, establishing a hierarchy (`novell` and `tech` are children of `book`; `fiction` is a child of `novell`).  
   - Persists manufacturers (`oreilley`, `packed`, `novells`).  
   - Creates a single product, associates it with `oreilley`, the `tech` category, and assigns price, availability, and a description.  
   - Saves the product through `productService`.  
   - Performs a few queries (listing, counting by category, retrieving manufacturers).  
   - Updates pricing (both existing and new “eco” price).  
   - Calls helper methods to test attributes, images, and finally deletes the product.  
3. **Helper Methods** –  
   - `testAttributes` creates product options, option values, and attributes.  
   - `testInsertImage` loads an image from the classpath, stores it via `productService`.  
   - `testViewImage` verifies that the image can be retrieved in both small and large sizes.  
   - `testReview` and `testCreateRelationShip` are defined but never invoked.  

### 2.2 Assumptions & Constraints  

- The default store and language codes already exist in the database.  
- The product, category, manufacturer, and image tables are empty or accept duplicate codes (the test doesn’t handle conflicts).  
- The test runs in a single transaction that is **committed** (no rollback) – leaving residual data.  
- The file system location for images is writable by the test process.  
- No concurrency or parallel execution is considered; the test is **stateful**.

### 2.3 Architecture & Design Choices  

The test follows an imperative, “cookbook” style: each entity is built inline, persisted, and then verified. While this makes the flow obvious, it also leads to:

- **Code duplication** – e.g., creation of categories and manufacturers is repeated with only minor differences.  
- **Hard‑coded strings** – codes, names, and file names are littered throughout the method.  
- **Missing assertions** – many operations are performed without verifying that the state changed as expected.  
- **No cleanup** – deleted only at the end of the test, but any failure before deletion leaves orphaned data.  

The use of `java.sql.Date` for availability dates, while fine, ties the code to the old SQL API rather than `java.time`.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `testCreateProduct()` | Main integration test that exercises product creation, retrieval, update, and deletion. | None (uses injected services). | `void` | Persists categories, manufacturers, product, images, and attributes; deletes the product at the end. |
| `testAttributes(Product)` | Creates product options/values/attributes for a given product. | `Product` instance | `void` | Persists options, values, and attributes; updates the product. |
| `testInsertImage(Product)` | Loads an image from the classpath and associates it with a product. | `Product` instance | `void` | Persists `ProductImage`; writes the file to disk. |
| `testViewImage(Product)` | Retrieves the stored image in small and large sizes and asserts non‑null. | `Product` instance | `void` | Reads files from disk via `productImageService`. |
| `testReview(Product)` | Creates a review for a product (not invoked). | `Product` instance | `void` | Persists a `ProductReview`. |
| `testCreateRelationShip(Product)` | Creates a related product and associates it with the given product (not invoked). | `Product` instance | `void` | Persists a related `Product` and a `ProductRelationship`. |

The test class contains no reusable utility methods; all domain‑specific logic is inline.

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| **Spring Framework** | Third‑party | Provides dependency injection, transaction management, and JUnit integration. |
| **JPA / Hibernate** | Third‑party | ORM for persisting domain entities. |
| **JUnit 4** | Third‑party | Test framework. |
| **SalesManager Core libraries** (`productService`, `categoryService`, etc.) | In‑house | Business services and domain model. |
| **Java Standard Library** | Standard | Collections, I/O, dates, BigDecimal. |
| **File System** | Platform‑specific | Storage of image files. |

No external REST or messaging systems are involved; all interactions are local via Spring beans.

---

## 5. Additional Notes & Recommendations  

### 5.1 Edge Cases & Potential Failures  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Duplicate code handling** – Categories and manufacturers are created without checking if the code already exists. | If the test runs more than once against the same database, `UniqueConstraintViolation` will occur. | Add a lookup before creation or generate a random suffix for the code. |
| **Unclosed InputStream** – `testInsertImage` never closes the stream. | Minor memory leak; in a real test environment could exhaust file handles. | Use `try (InputStream is = …) { … }` or let Spring close it. |
| **Hard‑coded paths & names** – `"img/" + IMAGE_NAME` and hard‑coded codes. | Brittle when refactoring or running in different environments. | Move to constants or configuration properties. |
| **Lack of assertions** – Most steps only call services without verifying outcomes. | The test may pass even if the persistence layer fails silently. | Add assertions after each persistence call (e.g., `Assert.assertNotNull(product.getId());`). |
| **No cleanup on failure** – The `productService.delete(refreshed)` is only executed if everything succeeds. | Partial runs leave dangling data, causing flaky subsequent tests. | Use `@After` to delete any created entities, or run the test in a transactional context that rolls back. |
| **Date handling** – Uses `java.sql.Date` with a static value. | All products share the same availability date; not realistic. | Use `LocalDate.now()` or a dynamic value. |
| **Thread safety** – The static `date` field is shared across tests. | In parallel execution, the same date is reused; may cause subtle bugs. | Generate dates per test invocation. |
| **Ignored helper methods** – `testReview` and `testCreateRelationShip` are defined but never used. | Dead code may mislead readers. | Either invoke them or remove. |
| **Hard‑coded `ProductOptionType.Radio.name()`** – Using string representation rather than enum. | Potential typo errors. | Pass the enum directly (`ProductOptionType.Radio`). |
| **Category `tech` descriptions** – Adds `techFrenchDescription` twice, missing English. | Incorrect multilingual data. | Add `techEnglishDescription` instead. |

### 5.2 Possible Enhancements  

1. **Use a builder pattern** – Create reusable builders for `Category`, `Product`, etc., to reduce boilerplate.  
2. **Extract helper methods** – `createCategory(String code, Map<Language,String> names)`, `createManufacturer(String code, String name)`, etc.  
3. **Transactional test** – Annotate the test with `@Transactional` and rely on Spring’s rollback to keep the database clean.  
4. **Parameterized tests** – Use JUnit’s `@RunWith(Parameterized.class)` to test multiple category hierarchies or product attributes.  
5. **Integration test framework** – Use Spring Boot Test or Arquillian for more declarative tests.  
6. **Assertions** – Validate that created entities have IDs, correct relationships, and that image files actually exist on disk.  
7. **Logging** – Replace `System.out.println` with a logger (`SLF4J`).  
8. **Use `LocalDateTime`** – Modern date/time API.  
9. **Avoid duplicate code** – Centralize common logic, especially when adding child categories.  
10. **Error handling** – Wrap persistence calls in try/catch to log detailed errors if something fails.  

---

## 6. Conclusion  

`ProductTest` demonstrates a complete product lifecycle in the SalesManager core domain but suffers from **imperative duplication, lack of assertions, hard‑coded values, and resource leaks**. By refactoring into reusable builders, adding proper cleanup, and enforcing assertions, the test suite will become **robust, maintainable, and easier to extend**.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.catalog;

import java.io.InputStream;
import java.math.BigDecimal;
import java.sql.Date;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import org.junit.Assert;
import org.junit.Test;
import com.salesmanager.core.business.exception.ServiceException;
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
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.manufacturer.ManufacturerDescription;
import com.salesmanager.core.model.catalog.product.price.ProductPrice;
import com.salesmanager.core.model.catalog.product.price.ProductPriceDescription;
import com.salesmanager.core.model.catalog.product.relationship.ProductRelationship;
import com.salesmanager.core.model.catalog.product.review.ProductReview;
import com.salesmanager.core.model.catalog.product.review.ProductReviewDescription;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;


public class ProductTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	
	private static final Date date = new Date(System.currentTimeMillis());
	
	private final String IMAGE_NAME = "icon.png";

	/**
	 * This method creates multiple products using multiple catalog APIs
	 * @throws ServiceException
	 */
	@Test
	public void testCreateProduct() throws Exception {

	    Language en = languageService.getByCode("en");
	    Language fr = languageService.getByCode("fr");

	    MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
	    ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);

	    Category book = new Category();
	    book.setMerchantStore(store);
	    book.setCode("book");

	    CategoryDescription bookEnglishDescription = new CategoryDescription();
	    bookEnglishDescription.setName("Book");
	    bookEnglishDescription.setCategory(book);
	    bookEnglishDescription.setLanguage(en);

	    CategoryDescription bookFrenchDescription = new CategoryDescription();
	    bookFrenchDescription.setName("Livre");
	    bookFrenchDescription.setCategory(book);
	    bookFrenchDescription.setLanguage(fr);

	    Set<CategoryDescription> descriptions = new HashSet<CategoryDescription>();
	    descriptions.add(bookEnglishDescription);
	    descriptions.add(bookFrenchDescription);

	    book.setDescriptions(descriptions);

	    categoryService.create(book);

	    Category music = new Category();
	    music.setMerchantStore(store);
	    music.setCode("music");

	    CategoryDescription musicEnglishDescription = new CategoryDescription();
	    musicEnglishDescription.setName("Music");
	    musicEnglishDescription.setCategory(music);
	    musicEnglishDescription.setLanguage(en);

	    CategoryDescription musicFrenchDescription = new CategoryDescription();
	    musicFrenchDescription.setName("Musique");
	    musicFrenchDescription.setCategory(music);
	    musicFrenchDescription.setLanguage(fr);

	    Set<CategoryDescription> descriptions2 = new HashSet<CategoryDescription>();
	    descriptions2.add(musicEnglishDescription);
	    descriptions2.add(musicFrenchDescription);

	    music.setDescriptions(descriptions2);

	    categoryService.create(music);

	    Category novell = new Category();
	    novell.setMerchantStore(store);
	    novell.setCode("novell");

	    CategoryDescription novellEnglishDescription = new CategoryDescription();
	    novellEnglishDescription.setName("Novell");
	    novellEnglishDescription.setCategory(novell);
	    novellEnglishDescription.setLanguage(en);

	    CategoryDescription novellFrenchDescription = new CategoryDescription();
	    novellFrenchDescription.setName("Roman");
	    novellFrenchDescription.setCategory(novell);
	    novellFrenchDescription.setLanguage(fr);

	    Set<CategoryDescription> descriptions3 = new HashSet<CategoryDescription>();
	    descriptions3.add(novellEnglishDescription);
	    descriptions3.add(novellFrenchDescription);

	    novell.setDescriptions(descriptions3);
	    
	    novell.setParent(book);

	    categoryService.create(novell);
	    categoryService.addChild(book, novell);

	    Category tech = new Category();
	    tech.setMerchantStore(store);
	    tech.setCode("tech");

	    CategoryDescription techEnglishDescription = new CategoryDescription();
	    techEnglishDescription.setName("Technology");
	    techEnglishDescription.setCategory(tech);
	    techEnglishDescription.setLanguage(en);

	    CategoryDescription techFrenchDescription = new CategoryDescription();
	    techFrenchDescription.setName("Technologie");
	    techFrenchDescription.setCategory(tech);
	    techFrenchDescription.setLanguage(fr);

	    Set<CategoryDescription> descriptions4 = new HashSet<CategoryDescription>();
	    descriptions4.add(techFrenchDescription);
	    descriptions4.add(techFrenchDescription);

	    tech.setDescriptions(descriptions4);
	    
	    tech.setParent(book);

	    categoryService.create(tech);
	    categoryService.addChild(book, tech);

	    Category fiction = new Category();
	    fiction.setMerchantStore(store);
	    fiction.setCode("fiction");

	    CategoryDescription fictionEnglishDescription = new CategoryDescription();
	    fictionEnglishDescription.setName("Fiction");
	    fictionEnglishDescription.setCategory(fiction);
	    fictionEnglishDescription.setLanguage(en);

	    CategoryDescription fictionFrenchDescription = new CategoryDescription();
	    fictionFrenchDescription.setName("Sc Fiction");
	    fictionFrenchDescription.setCategory(fiction);
	    fictionFrenchDescription.setLanguage(fr);

	    Set<CategoryDescription> fictiondescriptions = new HashSet<CategoryDescription>();
	    fictiondescriptions.add(fictionEnglishDescription);
	    fictiondescriptions.add(fictionFrenchDescription);

	    fiction.setDescriptions(fictiondescriptions);
	    
	    fiction.setParent(novell);

	    categoryService.create(fiction);
	    categoryService.addChild(book, fiction);

	    Manufacturer oreilley = new Manufacturer();
	    oreilley.setMerchantStore(store);
	    oreilley.setCode("oreilley");

	    ManufacturerDescription oreilleyd = new ManufacturerDescription();
	    oreilleyd.setLanguage(en);
	    oreilleyd.setName("O\'reilley");
	    oreilleyd.setManufacturer(oreilley);
	    oreilley.getDescriptions().add(oreilleyd);

	    manufacturerService.create(oreilley);

	    Manufacturer packed = new Manufacturer();
	    packed.setMerchantStore(store);
	    packed.setCode("packed");

	    ManufacturerDescription packedd = new ManufacturerDescription();
	    packedd.setLanguage(en);
	    packedd.setManufacturer(packed);
	    packedd.setName("Packed publishing");
	    packed.getDescriptions().add(packedd);

	    manufacturerService.create(packed);

	    Manufacturer novells = new Manufacturer();
	    novells.setMerchantStore(store);
	    novells.setCode("novells");

	    ManufacturerDescription novellsd = new ManufacturerDescription();
	    novellsd.setLanguage(en);
	    novellsd.setManufacturer(novells);
	    novellsd.setName("Novells publishing");
	    novells.getDescriptions().add(novellsd);

	    manufacturerService.create(novells);

	    // PRODUCT 1

	    Product product = new Product();
	    product.setProductHeight(new BigDecimal(4));
	    product.setProductLength(new BigDecimal(3));
	    product.setProductWidth(new BigDecimal(1));
	    product.setSku("CT12345");
	    product.setManufacturer(oreilley);
	    product.setType(generalType);
	    product.setMerchantStore(store);

	    // Product description
	    ProductDescription description = new ProductDescription();
	    description.setName("Spring in Action");
	    description.setLanguage(en);
	    description.setProduct(product);

	    product.getDescriptions().add(description);

	    //add category
	    product.getCategories().add(tech);

	    

	    // Availability
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(date);
	    availability.setProductQuantity(100);
	    availability.setRegion("*");
	    availability.setProduct(product);// associate with product

	    //productAvailabilityService.create(availability);
	    product.getAvailabilities().add(availability);

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
	    
	    
	    /**
	     * Create product
	     */
	    productService.saveProduct(product);
	    
	    
	    /**
	     * Creates a review
	     * TODO requires customer
	     */
	    //testReview(product);

	    List<Product> products = productService.listByStore(store);
	    
	    System.out.println("Total number of items " + products.size());
	    
	    //count products by category
		String lineage = new StringBuilder().append(book.getLineage()).toString();
		
		List<Category> categories = categoryService.getListByLineage(store, lineage);
		
		List<Long> ids = new ArrayList<Long>();
		if(categories!=null && categories.size()>0) {
			for(Category c : categories) {
				ids.add(c.getId());
			}
		} 
		
		List<Object[]> objs = categoryService.countProductsByCategories(store, ids);

		for(Object[] ob : objs) {
			Long c = (Long) ob[0];
			//System.out.println("Category " + c.getCode() + " has " + ob[1] + " items");
		}

		//get manufacturer for given categories
		List<Manufacturer> manufacturers = manufacturerService.listByProductsByCategoriesId(store, ids, en);
	    
		//System.out.println("Number of manufacturer for all category " + manufacturers.size());
		
		//Update product -- get first from the list
		Product updatableProduct = products.get(0);

		//Get first availability, which is the only one created
		ProductAvailability updatableAvailability = updatableProduct.getAvailabilities().iterator().next();
		
		//Get first price, which is the only one created
		ProductPrice updatablePrice = updatableAvailability.getPrices().iterator().next();
		updatablePrice.setProductPriceAmount(new BigDecimal(19.99));
		
		
		//Add a second price
	    ProductPrice anotherPrice = new ProductPrice();
	    anotherPrice.setCode("eco");
	    anotherPrice.setProductPriceAmount(new BigDecimal(1.99));
	    anotherPrice.setProductAvailability(updatableAvailability);

	    ProductPriceDescription anotherPriceD = new ProductPriceDescription();
	    anotherPriceD.setName("Eco price");
	    anotherPriceD.setProductPrice(anotherPrice);
	    anotherPriceD.setLanguage(en);

	    anotherPrice.getDescriptions().add(anotherPriceD);
	    updatableAvailability.getPrices().add(anotherPrice);
		
		//Update product
		productService.update(updatableProduct);
		
		
		//go and get products again
		products = productService.listByStore(store);

		updatableProduct = products.get(0);
		
		//test attributes
		this.testAttributes(updatableProduct);
		
		
		//test insert, view image
		testInsertImage(updatableProduct);
		testViewImage(updatableProduct);
		
		Product refreshed = productService.getBySku("CT12345", store, en);
		productService.delete(refreshed);

		
	    
	}

	
	private void testAttributes(Product product) throws Exception {
		
		
		/**
		 * An attribute can be created dynamicaly but the attached Option and Option value need to exist
		 */
		
		MerchantStore store = product.getMerchantStore();
		
		Language en = languageService.getByCode("en");
		
	    /**
	     * Create size option
	     */
	    ProductOption color = new ProductOption();
	    color.setMerchantStore(store);
	    color.setCode("COLOR");
	    color.setProductOptionType(ProductOptionType.Radio.name());
	    
	    ProductOptionDescription optionDescription = new ProductOptionDescription();
	    optionDescription.setLanguage(en);
	    optionDescription.setName("Color");
	    optionDescription.setDescription("Color of an item");
	    optionDescription.setProductOption(color);
	    
	    color.getDescriptions().add(optionDescription);
	    
	    //create option
	    productOptionService.saveOrUpdate(color);
	    
	     /**
         * Create size option
         */
        ProductOption size = new ProductOption();
        size.setMerchantStore(store);
        size.setCode("SIZE");
        size.setProductOptionType(ProductOptionType.Radio.name());
        
        ProductOptionDescription sizeDescription = new ProductOptionDescription();
        sizeDescription.setLanguage(en);
        sizeDescription.setName("Size");
        sizeDescription.setDescription("Size of an item");
        sizeDescription.setProductOption(size);
        
        size.getDescriptions().add(sizeDescription);
        
        //create option
        productOptionService.saveOrUpdate(size);
	    
	    
	    //option value
	    ProductOptionValue red = new ProductOptionValue();
	    red.setMerchantStore(store);
	    red.setCode("red");
	    
	    ProductOptionValueDescription redDescription = new ProductOptionValueDescription();
	    redDescription.setLanguage(en);
	    redDescription.setName("Red");
	    redDescription.setDescription("Red color");
	    redDescription.setProductOptionValue(red);
	    
	    red.getDescriptions().add(redDescription);
	    
	    //create an option value
	    productOptionValueService.saveOrUpdate(red);
	    
	    //another option value
	    ProductOptionValue blue = new ProductOptionValue();
	    blue.setMerchantStore(store);
	    blue.setCode("blue");
	    
	    ProductOptionValueDescription blueDescription = new ProductOptionValueDescription();
	    blueDescription.setLanguage(en);
	    blueDescription.setName("Blue");
	    blueDescription.setDescription("Color blue");
	    blueDescription.setProductOptionValue(blue);
	    
	    blue.getDescriptions().add(blueDescription);

	    //create another option value
	    productOptionValueService.saveOrUpdate(blue);
	    
	    //option value
        ProductOptionValue small = new ProductOptionValue();
        small.setMerchantStore(store);
        small.setCode("small");
        
        ProductOptionValueDescription smallDescription = new ProductOptionValueDescription();
        smallDescription.setLanguage(en);
        smallDescription.setName("Small");
        smallDescription.setDescription("Small size");
        smallDescription.setProductOptionValue(small);
        
        small.getDescriptions().add(smallDescription);
        
        //create an option value
        productOptionValueService.saveOrUpdate(small);
        
        //another option value
        ProductOptionValue medium = new ProductOptionValue();
        medium.setMerchantStore(store);
        medium.setCode("medium");
        
        ProductOptionValueDescription mediumDescription = new ProductOptionValueDescription();
        mediumDescription.setLanguage(en);
        mediumDescription.setName("Medium");
        mediumDescription.setDescription("Medium size");
        mediumDescription.setProductOptionValue(medium);
        
        medium.getDescriptions().add(mediumDescription);

        //create another option value
        productOptionValueService.saveOrUpdate(medium);
        
        
        ProductAttribute color_blue = new ProductAttribute();
        color_blue.setProduct(product);
        color_blue.setProductOption(color);
        color_blue.setAttributeDefault(true);
        color_blue.setProductAttributePrice(new BigDecimal(0));//no price variation
        color_blue.setProductAttributeWeight(new BigDecimal(1));//weight variation
        color_blue.setProductOptionValue(blue);
        
        productAttributeService.create(color_blue);
        
        product.getAttributes().add(color_blue);
        
	    
	    /** create attributes **/
	    //attributes
	    ProductAttribute color_red = new ProductAttribute();
	    color_red.setProduct(product);
	    color_red.setProductOption(color);
	    color_red.setAttributeDefault(true);
	    color_red.setProductAttributePrice(new BigDecimal(0));//no price variation
	    color_red.setProductAttributeWeight(new BigDecimal(1));//weight variation
	    color_red.setProductOptionValue(red);
	    
	    productAttributeService.create(color_red);
	    
	    product.getAttributes().add(color_red);


	    ProductAttribute smallAttr = new ProductAttribute();
	    smallAttr.setProduct(product);
	    smallAttr.setProductOption(size);
	    smallAttr.setAttributeDefault(true);
	    smallAttr.setProductAttributePrice(new BigDecimal(0));//no price variation
	    smallAttr.setProductAttributeWeight(new BigDecimal(1));//weight variation
	    smallAttr.setProductOptionValue(small);
	    
	    productAttributeService.create(smallAttr);
	    
	    product.getAttributes().add(smallAttr);
	    
	    productService.update(product);
	    
	    /**
	     * get options facets
	     */
	    
	    List<ProductAttribute> attributes = productAttributeService.getProductAttributesByCategoryLineage(store, product.getCategories().iterator().next().getLineage(), en);
	    Assert.assertTrue((long) attributes.size() > 0);

	}
	
	/**
	 * Images
	 * @param product
	 * @throws Exception
	 */
	private void testInsertImage(Product product) throws Exception {
		
		
		ProductImage productImage = new ProductImage();
		
		ClassLoader classloader = Thread.currentThread().getContextClassLoader();
		InputStream inputStream = classloader.getResourceAsStream("img/" + IMAGE_NAME);

        
        ImageContentFile cmsContentImage = new ImageContentFile();
        cmsContentImage.setFileName( IMAGE_NAME );
        cmsContentImage.setFile( inputStream );
        cmsContentImage.setFileContentType(FileContentType.PRODUCT);
        

        productImage.setProductImage(IMAGE_NAME);
        productImage.setProduct(product);
        
        //absolutely required otherwise the file is not created on disk
        productImage.setImage(inputStream);
        
        product.getImages().add(productImage);
        
        productService.update(product);//saves the ProductImage entity and the file on disk
        
        
		
		
	}
	
	private void testViewImage(Product product) throws Exception {
		
		
		ProductImage productImage = product.getProductImage();

        //get physical small image
        OutputContentFile contentFile = productImageService.getProductImage(product.getMerchantStore().getCode(), product.getSku(), productImage.getProductImage(), ProductImageSize.SMALL);
        
        Assert.assertNotNull(contentFile);

   	 	//get physical original image
        contentFile = productImageService.getProductImage(product.getMerchantStore().getCode(), product.getSku(), productImage.getProductImage(), ProductImageSize.LARGE);
        
        Assert.assertNotNull(contentFile);

		
	}
	
	
	//REVIEW
	private void testReview(Product product) throws Exception {
	  
	     ProductReview review = new ProductReview();
	     review.setProduct(product);
	     review.setReviewRating(4d);
	     Language en = languageService.getByCode("en");
	        
	     ProductReviewDescription reviewDescription = new ProductReviewDescription();
	     reviewDescription.setLanguage(en);
	     reviewDescription.setDescription("This is a product review");
	     reviewDescription.setName("A review for you");
	     reviewDescription.setProductReview(review);
	     review.getDescriptions().add(reviewDescription);
	        
	     productReviewService.create(review);
	  
	}
	
	private void testCreateRelationShip(Product product) throws Exception {
		
		MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
		Language en = languageService.getByCode("en");
		Manufacturer oreilley = manufacturerService.getByCode(store, "oreilley");
		ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);
		
		Category tech = categoryService.getByCode(store, "tech");
		
		
		//create new related product
	    // PRODUCT 1

	    Product related = new Product();
	    related.setProductHeight(new BigDecimal(4));
	    related.setProductLength(new BigDecimal(3));
	    related.setProductWidth(new BigDecimal(1));
	    related.setSku("TB67891");
	    related.setManufacturer(oreilley);
	    related.setType(generalType);
	    related.setMerchantStore(store);

	    // Product description
	    ProductDescription description = new ProductDescription();
	    description.setName("Spring 4 in Action");
	    description.setLanguage(en);
	    description.setProduct(related);

	    product.getDescriptions().add(description);

	    //add category
	    product.getCategories().add(tech);

	    

	    // Availability
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(date);
	    availability.setProductQuantity(200);
	    availability.setRegion("*");
	    availability.setProduct(related);// associate with product

	    //productAvailabilityService.create(availability);
	    related.getAvailabilities().add(availability);

	    ProductPrice dprice = new ProductPrice();
	    dprice.setDefaultPrice(true);
	    dprice.setProductPriceAmount(new BigDecimal(39.99));
	    dprice.setProductAvailability(availability);

	    ProductPriceDescription dpd = new ProductPriceDescription();
	    dpd.setName("Base price");
	    dpd.setProductPrice(dprice);
	    dpd.setLanguage(en);

	    dprice.getDescriptions().add(dpd);
	    availability.getPrices().add(dprice);
	    
	    related.getAvailabilities().add(availability);
	    
	    productService.save(related);
	    
	    ProductRelationship relationship = new ProductRelationship();
	    
	    relationship.setActive(true);
	    relationship.setCode("spring");
	    relationship.setProduct(product);
	    relationship.setRelatedProduct(related);
	    relationship.setStore(store);
	    
	    
	    //because relationships are nor joined fetched, make sure you query relationships first, then ad to an existing list
	    //so relationship and review are they only objects not joined fetch when querying a product
	    //need to do a subsequent query
	    List<ProductRelationship> relationships = productRelationshipService.listByProduct(product);
	    
	    
	    relationships.add(relationship);
	    
	    product.setRelationships(new HashSet<ProductRelationship>(relationships));
	    
	    productService.save(product);
		
		
	}
	




}


```
