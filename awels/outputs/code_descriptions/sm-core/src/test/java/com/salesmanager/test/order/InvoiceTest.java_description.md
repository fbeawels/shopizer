# InvoiceTest.java

## Review

## 1. Summary  

The file is a skeleton for a JUnit integration test that was meant to create a complete order (products, customer, billing, etc.), generate an OpenDocument Spreadsheet (ODS) invoice template, populate it with data, and finally convert the sheet to a PDF.  

**Key components**  

| Component | Role |
|-----------|------|
| `MerchantStore`, `Product`, `Order`, `OrderProduct`, `OrderTotal`, … | Domain entities that model a commerce system. |
| `productService`, `customerService`, `orderService` | Service layers that persist the domain objects. |
| `ProductPriceUtils` | Utility for formatting prices according to a store’s locale/currency. |
| `OOUtils`, `OpenDocument`, `ODTRenderer` | Apache OpenOffice / ODF toolkit for reading/writing ODS files. |
| `iText` classes (`Document`, `PdfWriter`, `PdfTemplate`, …) | PDF rendering library used to convert the ODS into a PDF. |

The test is heavily annotated with `@Ignore`, and the entire body of the test (`createInvoice`) is commented out, which means the test never executes.

The design is typical of an integration test that exercises multiple layers (service, persistence, file I/O), but the current implementation is incomplete, un‑tested, and hard to maintain.

---

## 2. Detailed Description  

### Flow of execution (as intended)  

1. **Setup services and utilities** – Dependency injection of all service objects and the price‑utility.  
2. **Create a product** – Build a `Product` with options, prices, attributes, and persist it.  
3. **Create a customer** – Construct billing & delivery addresses, persist the customer.  
4. **Build an order** – Attach the product, apply totals, set statuses, and persist the order.  
5. **Load an ODS template** – Use the ODF toolkit to read `invoice.ods`.  
6. **Populate the spreadsheet** – Write store info, customer info, product list, and totals into the sheet.  
7. **Save the ODS** – Write the modified sheet to a temporary file.  
8. **Convert to PDF** – Load the ODS again, render it onto a PDF canvas via iText, and write a PDF file.  
9. **Cleanup** – Delete the temporary ODS file.

### Assumptions & constraints  

| Assumption | Why it matters |
|------------|----------------|
| The database is empty before the test runs. | The test relies on `orderService.count() == 1` to confirm creation. |
| The file `templates/invoice/invoice.ods` exists in the classpath. | A missing template would cause a `FileNotFoundException`. |
| All services (`merchantService`, `productService`, etc.) are correctly wired via Spring. | The test is a Spring integration test; failing to wire will cause `NullPointerException`. |
| Locale and currency formatting logic in `ProductPriceUtils` is robust for all locales. | Formatting errors will surface in the generated PDF. |
| The target environment has LibreOffice/ODF dependencies available for `ODTRenderer`. | Without the required native libraries, rendering will fail. |

### Design choices  

* **Monolithic test** – The test does a lot of heavy lifting (entity creation + file generation) in a single method, which makes it hard to isolate failures.  
* **Hard‑coded paths & strings** – The test uses literal file paths (`"templates/invoice/invoice.ods"`) and hard‑coded customer data, reducing re‑usability.  
* **Manual PDF conversion** – The test performs manual PDF rendering via iText rather than delegating to a higher‑level service.  
* **Ignored and commented out** – The entire logic is commented out, indicating that it was either a draft or a work‑in‑progress.

---

## 3. Functions/Methods  

Only one public method exists, `createInvoice()`. It is currently `@Ignore`d and the body is commented out.  
Below is a high‑level description of the operations that would be performed if the method were active:

| Section | What it does | Inputs | Outputs / Side effects |
|---------|--------------|--------|------------------------|
| **Product creation** | Builds a `Product` with dimensions, SKU, price, option & attribute. | None (hard‑coded values) | Persists product via `productService.createProduct`. |
| **Customer creation** | Constructs `Customer`, `Billing`, `Delivery` objects. | None (hard‑coded values) | Persists customer via `customerService.create`. |
| **Order creation** | Creates an `Order` linked to the customer and product, attaches order totals, status history, and order products. | None (uses created entities) | Persists order via `orderService.create`. |
| **ODS template loading** | Reads `/templates/invoice/invoice.ods`. | File path | `SpreadSheet` object. |
| **Populate spreadsheet** | Writes store, customer, product, and totals information into specific cells. | `Order`, `MerchantStore`, `Customer` | Mutates the `SpreadSheet`. |
| **ODS output** | Saves the mutated spreadsheet to a temporary file. | `File` name | Temporary ODS file. |
| **PDF rendering** | Loads the ODS, uses `ODTRenderer` to paint onto an iText PDF canvas, writes PDF. | `order.getId() + "_invoice.ods"`, output PDF name | PDF file on disk. |
| **Cleanup** | Deletes the temporary ODS file. | None | Removes temp file. |

There are no reusable helper methods; all logic is inline, making it difficult to unit‑test small pieces.

---

## 4. Dependencies  

| Dependency | Nature | Notes |
|------------|--------|-------|
| **JUnit 4** (`@Test`, `@Ignore`) | Standard | Used for test orchestration. |
| **Spring (or similar)** (`@Inject`) | Third‑party | Provides services and utilities. |
| **Apache OpenOffice / ODF Toolkit** (`OpenDocument`, `ODTRenderer`, `SpreadSheet`) | Third‑party | Reads/writes ODS files. |
| **iText** (`Document`, `PdfWriter`, `PdfTemplate`, …) | Third‑party | Renders PDF from graphics. |
| **Commons‑Lang** (`StringUtils`) | Third‑party | Utility for string handling. |
| **Joda‑Time / Java 8 Time** (implicit via `Date`, `Locale`) | Standard | For date formatting. |
| **SalesManager domain models** (`MerchantStore`, `Product`, `Order`, …) | Internal | Business entities. |
| **Utility classes** (`ProductPriceUtils`, `OOUtils`) | Internal | Custom helper methods. |

No platform‑specific dependencies are obvious, but the ODF rendering may require native LibreOffice binaries on the classpath, which could be an issue on some CI environments.

---

## 5. Additional Notes & Recommendations  

### 5.1 Why the test is ineffective  
* **Entire body commented out** – The test does nothing, so it does not validate any functionality.  
* **No assertions on the PDF** – Even if the PDF were generated, there are no checks that the contents are correct.  
* **Hard‑coded data** – Any change in business rules (e.g., new mandatory fields) would require editing the test.  

### 5.2 Suggested refactorings  

| Area | Recommendation |
|------|----------------|
| **Split responsibilities** | Separate data creation (fixtures) from invoice generation. Create utility methods or a test fixture builder (`OrderTestDataBuilder`). |
| **Use Spring TestContext** | Annotate the class with `@RunWith(SpringJUnit4ClassRunner.class)` and `@ContextConfiguration` so that dependency injection works automatically. |
| **Parameterize data** | Use `@Parameterized` or external CSV/JSON to drive test data, improving maintainability. |
| **Mock file I/O** | Replace real file writes with temporary directories or in‑memory streams (e.g., `java.nio.file.Files.createTempFile`) to avoid side effects. |
| **Assert on PDF content** | After PDF generation, use a PDF parsing library (e.g., Apache PDFBox) to extract text and verify key fields (invoice number, customer name, totals). |
| **Move PDF rendering to a service** | Encapsulate the ODS→PDF logic in a service class (e.g., `InvoicePdfGenerator`) and unit‑test that class separately. |
| **Use `@After` cleanup** | Ensure temporary files are deleted even when the test fails. |
| **Remove magic numbers** | The cell coordinates (e.g., `0, 0`, `3, 2`) should be constants or defined in a template configuration. |

### 5.3 Edge cases / missing scenarios  

* **Locale differences** – The test only uses English; adding tests for other locales would surface formatting issues.  
* **Large orders** – No test for pagination or overflow when there are many products.  
* **Error handling** – The `try` block swallows all exceptions; the test should fail loudly if anything goes wrong.  
* **Resource leaks** – Streams (e.g., `FileOutputStream`) are not closed in a `finally` block.  

### 5.4 Future enhancements  

1. **Externalize template** – Store ODS templates in a database or external storage so that the test can load them programmatically.  
2. **Template engine** – Replace the manual cell writing with a template engine (e.g., XDocReport) that maps data objects to placeholders.  
3. **Integration with CI** – Configure a Docker image that contains LibreOffice, so the PDF rendering can run on any CI runner.  
4. **Reporting** – Generate a summary report (e.g., JUnit XML or a simple Markdown) indicating which parts of the invoice matched expectations.  

---

### Bottom line  

The current `InvoiceTest` is a draft that demonstrates an approach but is not runnable. To become a valuable integration test, it needs to be fully implemented, cleaned up, and enriched with assertions. Breaking the test into smaller, reusable components and leveraging existing testing frameworks will greatly improve maintainability and reliability.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.order;

import org.junit.Ignore;


/**
 * This test has to be completed
 * @author c.samson
 *
 */
@Ignore
public class InvoiceTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
/*	
	@Inject
	ProductPriceUtils priceUtil;


	@Ignore
	public void createInvoice() throws ServiceException {
		

	    MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
	    
		//create a product
	    ProductType generalType = productTypeService.getProductType(ProductType.GENERAL_TYPE);
	    
	    Language en = languageService.getByCode("en");
	    
	    
	    *//**
	     * 1) Create an order
	     * 
	     *//*
	    
	    //1.1 create a product
	    
	    //create an option
	    ProductOption color = new ProductOption();
	    color.setMerchantStore(store);
	    color.setCode("color");
	    color.setProductOptionType("SELECT");
	    
	    ProductOptionDescription colorDescription = new ProductOptionDescription();
	    colorDescription.setDescription("Color");
	    colorDescription.setName("Color");
	    colorDescription.setLanguage(en);
	    colorDescription.setProductOption(color);
	    
	    Set<ProductOptionDescription> colorDescriptions = new HashSet<ProductOptionDescription>();
	    colorDescriptions.add(colorDescription);
	    
	    color.setDescriptions(colorDescriptions);
	    
	    productOptionService.create(color);
	    
	    //create an option value
	    ProductOptionValue red = new ProductOptionValue();
	    red.setMerchantStore(store);
	    red.setCode("red");
	    
	    ProductOptionValueDescription redDescription = new ProductOptionValueDescription();
	    redDescription.setDescription("Red");
	    redDescription.setLanguage(en);
	    redDescription.setName("Red");
	    redDescription.setProductOptionValue(red);
	    
	    Set<ProductOptionValueDescription> redDescriptions = new HashSet<ProductOptionValueDescription>();
	    redDescriptions.add(redDescription);
	    
	    red.setDescriptions(redDescriptions);
	    
	    productOptionValueService.create(red);

	    //create a product
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
	    

	    // Availability
	    ProductAvailability availability = new ProductAvailability();
	    availability.setProductDateAvailable(new Date());
	    availability.setProductQuantity(100);
	    availability.setRegion("*");
	    availability.setProduct(product);// associate with product
	    
	    product.getAvailabilities().add(availability);

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
	    
	    
	    //create an attribute
	    ProductAttribute colorAttribute = new ProductAttribute();
	    colorAttribute.setProduct(product);
	    colorAttribute.setProductAttributePrice(new BigDecimal(5));
	    colorAttribute.setProductOption(color);
	    colorAttribute.setProductOptionValue(red);
	    

	    product.getAttributes().add(colorAttribute);
	    
	    productService.createProduct(product);
	    

	    //1.2 create a Customer
		Country country = countryService.getByCode("CA");
		Zone zone = zoneService.getByCode("QC");
		
		Customer customer = new Customer();
		customer.setMerchantStore(store);
		customer.setEmailAddress("test@test.com");
		customer.setGender(CustomerGender.M);				
		customer.setAnonymous(true);
		customer.setCompany("ifactory");
		customer.setDateOfBirth(new Date());
		customer.setNick("My nick");
		customer.setDefaultLanguage(en);
		
		
	    Delivery delivery = new Delivery();
	    delivery.setAddress("358 Du Languadoc");
	    delivery.setCity( "Boucherville" );
	    delivery.setCountry(country);
//	    delivery.setCountryCode(CA_COUNTRY_CODE);
	    delivery.setFirstName("First" );
	    delivery.setLastName("Last" );
	    delivery.setPostalCode("J4B-8J9" );
	    delivery.setZone(zone);	    
	    
	    Billing billing = new Billing();
	    billing.setAddress("358 Du Languadoc");
	    billing.setCity("Boucherville");
	    billing.setCompany("CSTI Consulting");
	    billing.setCountry(country);
//	    billing.setCountryCode(CA_COUNTRY_CODE);
	    billing.setFirstName("Carl" );
	    billing.setLastName("Samson" );
	    billing.setPostalCode("J4B-8J9");
	    billing.setZone(zone);
	    
	    customer.setBilling(billing);
	    customer.setDelivery(delivery);		
		customerService.create(customer);
		
		Currency currency = currencyService.getByCode(CAD_CURRENCY_CODE);
		
		//1.3 create an order
		OrderStatusHistory orderStatusHistory = new OrderStatusHistory();
		
		
		Order order = new Order();
		order.setDatePurchased(new Date());
		order.setCurrency(currency);
		order.setLastModified(new Date());
		order.setBilling(billing);
		
		Locale l = Locale.CANADA;
		order.setLocale(l);


		order.setCurrencyValue(new BigDecimal(0.98));//compared to based currency (not necessary)
		order.setCustomerId(customer.getId());
		order.setDelivery(delivery);
		order.setIpAddress("ipAddress" );
		order.setMerchant(store);
		order.setCustomerEmailAddress(customer.getEmailAddress());
		order.setOrderDateFinished(new Date());//committed date
		
		orderStatusHistory.setComments("We received your order");
		orderStatusHistory.setCustomerNotified(1);
		orderStatusHistory.setStatus(OrderStatus.ORDERED);
		orderStatusHistory.setDateAdded(new Date() );
		orderStatusHistory.setOrder(order);
		order.getOrderHistory().add( orderStatusHistory );		
		

		order.setPaymentType(PaymentType.PAYPAL);
		order.setPaymentModuleCode("paypal");
		order.setStatus( OrderStatus.DELIVERED);
		order.setTotal(new BigDecimal(23.99));
		
		
		//OrderProductDownload - Digital download
		OrderProductDownload orderProductDownload = new OrderProductDownload();
		orderProductDownload.setDownloadCount(1);
		orderProductDownload.setMaxdays(31);		
		orderProductDownload.setOrderProductFilename("Your digital file name");
		
		//OrderProductPrice
		OrderProductPrice oproductprice = new OrderProductPrice();
		oproductprice.setDefaultPrice(true);	
		oproductprice.setProductPrice(new BigDecimal(19.99) );
		oproductprice.setProductPriceCode("baseprice" );
		oproductprice.setProductPriceName("Base Price" );

		//OrderProduct
		OrderProduct oproduct = new OrderProduct();
		oproduct.getDownloads().add( orderProductDownload);
		oproduct.setOneTimeCharge( new BigDecimal(19.99) );
		oproduct.setOrder(order);		
		oproduct.setProductName( "Product name" );
		oproduct.setProductQuantity(2);
		oproduct.setSku("TB12345" );		
		oproduct.getPrices().add(oproductprice ) ;
		
		
		//an attribute to the OrderProduct
		OrderProductAttribute orderAttribute = new OrderProductAttribute();
		orderAttribute.setOrderProduct(oproduct);
		orderAttribute.setProductAttributeName(colorDescription.getName());
		orderAttribute.setProductAttributeValueName(redDescription.getName());
		orderAttribute.setProductOptionId(color.getId());
		orderAttribute.setProductOptionValueId(red.getId());
		orderAttribute.setProductAttributePrice(colorAttribute.getProductAttributePrice());
		
		Set<OrderProductAttribute> orderAttributes = new HashSet<OrderProductAttribute>();
		orderAttributes.add(orderAttribute);
		
		oproduct.setOrderAttributes(orderAttributes);
		
		oproductprice.setOrderProduct(oproduct);		
		orderProductDownload.setOrderProduct(oproduct);
		order.getOrderProducts().add(oproduct);
		
		
		//product #2
		OrderProductPrice oproductprice2 = new OrderProductPrice();
		oproductprice2.setDefaultPrice(true);	
		oproductprice2.setProductPrice(new BigDecimal(9.99) );
		oproductprice2.setProductPriceCode("baseprice" );
		oproductprice2.setProductPriceName("Base Price" );

		//OrderProduct
		OrderProduct oproduct2 = new OrderProduct();
		oproduct2.setOneTimeCharge( new BigDecimal(9.99) );
		oproduct2.setOrder(order);		
		oproduct2.setProductName( "Additional item name" );
		oproduct2.setProductQuantity(1);
		oproduct2.setSku("TB12346" );		
		oproduct2.getPrices().add(oproductprice2 ) ;
		
		oproductprice2.setOrderProduct(oproduct2);		
		order.getOrderProducts().add(oproduct2);
		
		
		
		

		//requires 
		//OrderProduct
		//OrderProductPrice
		//OrderTotal
		

		
		//OrderTotal
		OrderTotal subtotal = new OrderTotal();	
		subtotal.setModule("summary" );		
		subtotal.setSortOrder(0);
		subtotal.setText("Summary" );
		subtotal.setTitle("Summary" );
		subtotal.setValue(new BigDecimal(19.99 ) );
		subtotal.setOrder(order);
		
		order.getOrderTotal().add(subtotal);
		
		OrderTotal tax = new OrderTotal();	
		tax.setModule("tax" );		
		tax.setSortOrder(1);
		tax.setText("Tax" );
		tax.setTitle("Tax" );
		tax.setValue(new BigDecimal(4) );
		tax.setOrder(order);
		
		order.getOrderTotal().add(tax);
		
		OrderTotal total = new OrderTotal();	
		total.setModule("total" );		
		total.setSortOrder(2);
		total.setText("Total" );
		total.setTitle("Total" );
		total.setValue(new BigDecimal(23.99) );
		total.setOrder(order);
		
		order.getOrderTotal().add(total);
		
		orderService.create(order);
		Assert.assertTrue(orderService.count() == 1);
		
		Locale locale = Locale.ENGLISH;
		
		
		order = orderService.getById(order.getId());
		
		*//**
		 * 2 Create an invoice
		 *//*
		try {
			URL resource = getClass().getResource("/templates/invoice/invoice.ods");
			File file = new File(resource.toURI());
			//File file = new File("templates/invoice/invoice.ods");
		
			Sheet sheet = SpreadSheet.createFromFile(file).getSheet(0);
			
			
			//Store name 
			sheet.setValueAt(store.getStorename(), 0, 0);
			
			store.setStoreaddress("2001 zoo avenue");
			store.setCurrencyFormatNational(true);//use $ instead of USD
			
			
			//Address
			//count store address cell
			int storeAddressCell = 2;
			//if(!StringUtils.isBlank(store.getStoreaddress())) {
			//	sheet.setValueAt(store.getStoreaddress(), 0, storeAddressCell);
			//	storeAddressCell ++;
			//}
			
			//3
			StringBuilder storeAddress = null;
			if(!StringUtils.isBlank(store.getStoreaddress())) {
				storeAddress = new StringBuilder();
				storeAddress.append(store.getStoreaddress());
			}
			if(!StringUtils.isBlank(store.getStorecity())) {
				if(storeAddress==null) {
					storeAddress = new StringBuilder();
				} else {
					storeAddress.append(", ");
				}
				storeAddress.append(store.getStorecity());
			}
			if(storeAddress!=null) {
				sheet.setValueAt(storeAddress.toString(), 0, storeAddressCell);
				storeAddressCell ++;
			}
			
			//4
			StringBuilder storeProvince = null;
			if(store.getZone()!=null) {
				storeProvince = new StringBuilder();
				List<Zone> zones = zoneService.getZones(store.getCountry(), en);
				for(Zone z : zones) {
					if(z.getCode().equals(store.getZone().getCode())) {
						storeProvince.append(store.getZone().getName());
						break;
					}
				}
				
			} else {
				if(!StringUtils.isBlank(store.getStorestateprovince())) {
					storeProvince = new StringBuilder();
					storeProvince.append(store.getStorestateprovince());
				}
			}
			if(store.getCountry()!=null) {
				if(storeProvince==null) {
					storeProvince = new StringBuilder();
				} else {
					storeProvince.append(", ");
				}
				Map<String,Country> countries = countryService.getCountriesMap(en);
				Country c = countries.get(store.getCountry().getIsoCode());
				if(c!=null) {
					storeProvince.append(c.getName());
				} else {
					storeProvince.append(store.getCountry().getIsoCode());
				}
				
			}
			if(storeProvince!=null) {
				sheet.setValueAt(storeProvince.toString(), 0, storeAddressCell);
				storeAddressCell ++;
			}
			
			//5
			if(!StringUtils.isBlank(store.getStorepostalcode())) {
				sheet.setValueAt(store.getStorepostalcode(), 0, storeAddressCell);
				storeAddressCell ++;
			}
			
			//6
			if(!StringUtils.isBlank(store.getStorephone())) {
				sheet.setValueAt(store.getStorephone(), 0, storeAddressCell);
			}
			
			//delete address blank lines
			for(int i = storeAddressCell; i<5; i++) {
				sheet.setValueAt("", 0, i);
			}

			//invoice date
			SimpleDateFormat format = new SimpleDateFormat(Constants.DEFAULT_DATE_FORMAT);
			sheet.setValueAt(format.format(order.getDatePurchased()), 3, 2);
			
			//invoice number
			sheet.setValueAt(order.getId(), 3, 3);
			
			//bill to
			//count bill to address cell
			int billToCell = 8;
			if(!StringUtils.isBlank(customer.getBilling().getFirstName())) {
				sheet.setValueAt(customer.getBilling().getFirstName() + " " + customer.getBilling().getLastName(), 0, billToCell);
				billToCell ++;
			}
			
			//9
			if(!StringUtils.isBlank(customer.getBilling().getCompany())) {
				sheet.setValueAt(customer.getBilling().getCompany(), 0, billToCell);
				billToCell ++;
			}
			
			//10
			StringBuilder billToAddress = null;
			if(!StringUtils.isBlank(customer.getBilling().getAddress())) {
				billToAddress = new StringBuilder();
				billToAddress.append(customer.getBilling().getAddress());
			}
			if(!StringUtils.isBlank(customer.getBilling().getCity())) {
				if(billToAddress==null) {
					billToAddress = new StringBuilder();
				} else {
					billToAddress.append(", ");
				}
				billToAddress.append(customer.getBilling().getCity());
			}
			if(billToAddress!=null) {
				sheet.setValueAt(billToAddress.toString(), 0, billToCell);
				billToCell ++;
			}
			
			//11
			StringBuilder billToProvince = null;
			if(customer.getBilling().getZone()!=null) {
				billToProvince = new StringBuilder();
				List<Zone> zones = zoneService.getZones(customer.getBilling().getCountry(), en);
				for(Zone z : zones) {
					if(z.getCode().equals(customer.getBilling().getZone().getCode())) {
						billToProvince.append(customer.getBilling().getZone().getName());
						break;
					}
				}
				
			} else {
				if(!StringUtils.isBlank(customer.getBilling().getState())) {
					billToProvince = new StringBuilder();
					billToProvince.append(customer.getBilling().getState());
				}
			}
			if(customer.getBilling().getCountry()!=null) {
				if(billToProvince==null) {
					billToProvince = new StringBuilder();
				} else {
					billToProvince.append(", ");
				}
				Map<String,Country> countries = countryService.getCountriesMap(en);
				Country c = countries.get(customer.getBilling().getCountry().getIsoCode());
				if(c!=null) {
					billToProvince.append(c.getName());
				} else {
					billToProvince.append(customer.getBilling().getCountry().getIsoCode());
				}
				
			}
			if(billToProvince!=null) {
				sheet.setValueAt(billToProvince.toString(), 0, billToCell);
				billToCell ++;
			}
			
			//12
			if(!StringUtils.isBlank(customer.getBilling().getPostalCode())) {
				sheet.setValueAt(customer.getBilling().getPostalCode(), 0, billToCell);
				billToCell ++;
			}
			
			//13
			if(!StringUtils.isBlank(customer.getBilling().getTelephone())) {
				sheet.setValueAt(customer.getBilling().getTelephone(), 0, billToCell);
			}
			
			//delete address blank lines
			for(int i = billToCell; i<13; i++) {
				sheet.setValueAt("", 0, i);
			}
			
			//products
			Set<OrderProduct> orderProducts = order.getOrderProducts();
			int productCell = 16;
			for(OrderProduct orderProduct : orderProducts) {
				
				//product name
				String pName = orderProduct.getProductName();
				Set<OrderProductAttribute> oAttributes = orderProduct.getOrderAttributes();
				StringBuilder attributeName = null;
				for(OrderProductAttribute oProductAttribute : oAttributes) {
					if(attributeName == null) {
						attributeName = new StringBuilder();
						attributeName.append("[");
					} else {
						attributeName.append(", ");
					}
					attributeName.append(oProductAttribute.getProductAttributeName())
					.append(": ")
					.append(oProductAttribute.getProductAttributeValueName());
					
				}
				
				
				StringBuilder productName = new StringBuilder();
				productName.append(pName);
				
				if(attributeName!=null) {
					attributeName.append("]");
					productName.append(" ").append(attributeName.toString());
				}
				
				
				
				
				sheet.setValueAt(productName.toString(), 0, productCell);
				
				int quantity = orderProduct.getProductQuantity();
				sheet.setValueAt(quantity, 1, productCell);
				String amount = priceUtil.getStoreFormatedAmountWithCurrency(store, orderProduct.getOneTimeCharge());
				sheet.setValueAt(amount, 2, productCell);
				String t = priceUtil.getStoreFormatedAmountWithCurrency(store, priceUtil.getOrderProductTotalPrice(store, orderProduct));
				sheet.setValueAt(t, 3, productCell);

				productCell++;
				
			}
			
			//print totals
			productCell++;
			Set<OrderTotal> totals = order.getOrderTotal();
			for(OrderTotal orderTotal : totals) {
				
				String totalName = orderTotal.getText();
				String totalValue = priceUtil.getStoreFormatedAmountWithCurrency(store,orderTotal.getValue());
				sheet.setValueAt(totalName, 2, productCell);
				sheet.setValueAt(totalValue, 3, productCell);
				productCell++;
			}
			
			//sheet.getCellAt(0, 0).setImage(arg0)
			//sheet.getCellAt(0, 0).setStyleName(arg0)
			//sheet.getCellAt(0, 0).getStyle().
			
			
			
			File outputFile = new File(order.getId() + "_invoice.ods");
			OOUtils.open(sheet.getSpreadSheet().saveAs(outputFile));
			
			
			final OpenDocument doc = new OpenDocument();
			doc.loadFrom(order.getId() + "_invoice.ods");

			 // Open the PDF document
			 Document document = new Document(PageSize.A4);
			 File outFile = new File("invoice.pdf");

			 PdfDocument pdf = new PdfDocument();

			 document.addDocListener(pdf);

			 FileOutputStream fileOutputStream = new FileOutputStream(outFile);
			 PdfWriter writer = PdfWriter.getInstance(pdf, fileOutputStream);
			 pdf.addWriter(writer);

			 document.open();

			 // Create a template and a Graphics2D object 
			 Rectangle pageSize = document.getPageSize();
			 int w = (int) (pageSize.getWidth() * 0.9);
			 int h = (int) (pageSize.getHeight() * 0.95);
			 PdfContentByte cb = writer.getDirectContent();
			 PdfTemplate tp = cb.createTemplate(w, h);

			 Graphics2D g2 = tp.createPrinterGraphics(w, h, null);
			 // If you want to prevent copy/paste, you can use
			 // g2 = tp.createGraphicsShapes(w, h, true, 0.9f);
			            
			 tp.setWidth(w);
			 tp.setHeight(h);

			 // Configure the renderer
			 ODTRenderer renderer = new ODTRenderer(doc);
			 renderer.setIgnoreMargins(true);
			 renderer.setPaintMaxResolution(true);
			            
			 // Scale the renderer to fit width
			 renderer.setResizeFactor(renderer.getPrintWidth() / w);
			 // Render
			 renderer.paintComponent(g2);
			 g2.dispose();

			 // Add our spreadsheet in the middle of the page
			 float offsetX = (pageSize.getWidth() - w) / 2;
			 float offsetY = (pageSize.getHeight() - h) / 2;
			 cb.addTemplate(tp, offsetX, offsetY);
			 // Close the PDF document
			 document.close();
			 
			 outputFile.delete();//remove temp file
			
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		
	}*/
	
	

}


```
