# ODSInvoiceModule.java

## Review

## 1. Summary

`ODSInvoiceModule` is a legacy implementation of the `InvoiceModule` interface that generates an invoice PDF from an OpenDocument Spreadsheet (ODS) template.  
The core responsibilities are:

| Responsibility | Where it lives | Key notes |
|----------------|----------------|-----------|
| Template selection (by locale) | `createInvoice()` (commented code) | Builds a path such as `templates/invoice/Invoice_en.ods` |
| Data population | `createInvoice()` | Fills store, billing address, product lines and totals into the first sheet |
| PDF rendering | `createInvoice()` | Uses a 3‑rd‑party ODT renderer (`ODTRenderer`) and iText to write the PDF |
| Resource cleanup | `createInvoice()` | Deletes temporary ODS and PDF files after rendering |

The code relies heavily on third‑party libraries: **JODConverter** (ODT/OpenDocument support), **iText** (PDF creation), and several custom services (`ZoneService`, `CountryService`, `ProductPriceUtils`). The implementation is currently commented out and the exposed method throws a `Not implemented` exception; this indicates a planned or legacy refactor.

---

## 2. Detailed Description

### Overall Flow (from the commented implementation)

1. **Template resolution**  
   - Build a locale‑specific template name (`Invoice_<lang>.ods`).  
   - Fallback to the default `Invoice.ods` if the locale one is missing.  
   - Throw an exception if no template can be loaded.

2. **File preparation**  
   - Copy the template from the classpath to a temporary file (`<orderId>_working.ods`).  
   - Load the first sheet using `SpreadSheet.createFromFile(file)`.

3. **Populate header data**  
   - Store name, address, and contact info are written to rows 0‑5.  
   - Invoice date and number are written to cells (3,2) and (3,3).

4. **Populate billing address**  
   - Similar logic to the store address, using `order.getBilling()`.

5. **Populate products**  
   - Iterate over `order.getOrderProducts()` writing name, quantity, unit price, and total.

6. **Populate totals**  
   - Iterate over `order.getOrderTotal()` writing line totals and subtotals.

7. **Generate ODS and convert to PDF**  
   - Save the modified ODS to a temp file (`<orderId>_invoice.ods`).  
   - Load the ODS with `OpenDocument`.  
   - Create an iText `Document` and `PdfWriter`, render the ODS using `ODTRenderer`, and write the PDF to a `ByteArrayOutputStream`.

8. **Cleanup**  
   - Delete the temporary ODS and working files.

### Dependencies & Assumptions

- **Locale handling** – The code assumes the language code matches a template file. No fallback mechanism beyond the default template.
- **File system** – Temporary files are created in the current working directory; no guarantee of a unique name or safe cleanup in concurrent environments.
- **Error handling** – Exceptions are thrown for missing resources but IO exceptions during read/write or rendering are only rethrown as generic `Exception`.
- **Third‑party libraries** – The implementation depends on JODConverter (for ODT), iText (for PDF), and possibly Apache Commons IO (`IOUtils`).

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `createInvoice(MerchantStore, Order, Language)` (deprecated) | Public contract of `InvoiceModule`. Currently throws `Not implemented`. | `store`, `order`, `language` | `ByteArrayOutputStream` | None – throws an exception. |
| (Commented) `createInvoice(...)` | The full ODS → PDF conversion logic. | Same as above | `ByteArrayOutputStream` | Creates temporary files, writes PDF bytes, deletes temp files. |

The only public method in the current class is the deprecated stub; all other logic is hidden in commented code, meaning the class currently has no functional behavior.

---

## 4. Dependencies

| Library | Purpose | Standard / Third‑party |
|---------|---------|------------------------|
| `javax.inject.Inject` | Dependency injection | Standard (JDK) |
| `org.slf4j.Logger` | Logging | Third‑party |
| `com.salesmanager.core.*` | Domain entities and services | Project‑specific |
| `JODConverter / OpenDocument` | Read/modify ODS | Third‑party |
| `iText` | PDF generation | Third‑party |
| `Apache Commons IO` (`IOUtils`) | Stream copying | Third‑party |
| `java.io.*`, `java.util.*`, `java.text.*` | Core Java APIs | Standard |

No platform‑specific APIs are used, but the code relies on the presence of a classpath‑based template and a writable file system.

---

## 5. Additional Notes & Recommendations

### 5.1. Why the implementation is commented out
- Likely a migration to a newer invoicing strategy (e.g., HTML → PDF).  
- The presence of a `@Deprecated` stub suggests the method is no longer used in production.

### 5.2. Edge‑cases & pitfalls

| Problem | Impact | Suggested Fix |
|---------|--------|---------------|
| **Duplicate temporary filenames** | Concurrent orders could clash, leading to corrupt PDFs. | Use `Files.createTempFile()` or append a UUID. |
| **Hard‑coded file paths** | May not have write permissions or may clutter the working dir. | Write to `java.io.tmpdir` or a configurable temp dir. |
| **Missing resource cleanup** | Files may linger if an exception occurs before deletion. | Use `try‑with‑resources` and finally blocks, or a `File.deleteOnExit()`. |
| **Inadequate error handling** | Generic `Exception` hides underlying causes (IO, parsing, rendering). | Catch specific exceptions, log details, and throw a domain‑specific `InvoiceGenerationException`. |
| **Unnecessary classpath resource duplication** | Copying the template to a new file may be redundant; JODConverter can open directly from an `InputStream`. | Open the ODS directly from the classpath InputStream, modify in memory, then write to temp file. |
| **Synchronous PDF rendering** | Blocks the calling thread; for high traffic this could become a bottleneck. | Consider async job queue or background worker. |

### 5.3. Design improvements

1. **Separation of concerns**  
   - **TemplateResolver** – Determines the correct ODS file.  
   - **DataBinder** – Populates a `Spreadsheet` with order data.  
   - **Renderer** – Converts a `Spreadsheet` to PDF.  
   - **FileManager** – Handles temporary files safely.

2. **Modern PDF libraries**  
   - Replace JODConverter/iText stack with a lightweight library (e.g., Apache PDFBox) or a templating engine (Thymeleaf + PDF) if the ODS route is no longer needed.

3. **Unit‑testability**  
   - Inject `FileSystem` abstraction to allow mocking of file operations.  
   - Expose a non‑deprecated `generatePdfFromSpreadsheet()` method for testing.

4. **Configuration**  
   - Move template directory, default language, and temp directory to configuration files or environment variables.

### 5.4. Future enhancements

- **Localization** – Support more than just the language code (e.g., country‑specific templates).
- **Styling** – Allow dynamic header/footer or branding per merchant store.
- **Streaming** – Return a `StreamingResponseBody` (Spring) or `InputStream` directly to avoid holding the entire PDF in memory.
- **Audit & Logging** – Log generation success/failure with order IDs for easier debugging.

---

### Bottom Line

`ODSInvoiceModule` is a legacy, incomplete implementation. The current public API is non‑functional, and the commented code demonstrates a classic “copy‑template → mutate → convert” pattern that suffers from hard‑coded paths, resource leaks, and poor error handling. If this module remains in the codebase, it should be refactored into a clean, testable, and maintainable component or replaced entirely with a modern PDF generation strategy.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.order;

import java.io.ByteArrayOutputStream;

import javax.inject.Inject;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.zone.ZoneService;
import com.salesmanager.core.business.utils.ProductPriceUtils;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.reference.language.Language;





public class ODSInvoiceModule implements InvoiceModule {
	
	private final static String INVOICE_TEMPLATE = "templates/invoice/Invoice";
	private final static String INVOICE_TEMPLATE_EXTENSION = ".ods";
	private final static String TEMP_INVOICE_SUFFIX_NAME = "_invoice.ods";
	private final static int ADDRESS_ROW_START = 2;
	private final static int ADDRESS_ROW_END = 5;
	
	private final static int BILLTO_ROW_START = 8;
	private final static int BILLTO_ROW_END = 13;
	
	private final static int PRODUCT_ROW_START = 16;
	
	private static final Logger LOGGER = LoggerFactory.getLogger( ODSInvoiceModule.class );
	
	@Inject
	private ZoneService zoneService;
	
	@Inject
	private CountryService countryService;
	
	@Inject
	private ProductPriceUtils priceUtil;

	@Deprecated
	@Override
	public ByteArrayOutputStream createInvoice(MerchantStore store, Order order, Language language) throws Exception {
		// TODO Auto-generated method stub
		throw new Exception("Not implemented");
	}
	

/*	@Override
	public ByteArrayOutputStream createInvoice(MerchantStore store, Order order, Language language) throws Exception {
		
		

			
			List<Zone> zones = zoneService.getZones(store.getCountry(), language);
			Map<String,Country> countries = countryService.getCountriesMap(language);
			
			//get default template
			String template = new StringBuilder().append(INVOICE_TEMPLATE).append("_").append(language.getCode().toLowerCase()).append(INVOICE_TEMPLATE_EXTENSION).toString();
			
			//try by language
			InputStream is = null;
			try {
				is = getClass().getClassLoader().getResourceAsStream(template);
			} catch (Exception e) {
				LOGGER.warn("Cannot open template " + template);
				throw new Exception("Cannot open " + template);
			}
			
			if(is==null) {
				try {
					is = getClass().getClassLoader().getResourceAsStream(new StringBuilder().append(INVOICE_TEMPLATE).append(INVOICE_TEMPLATE_EXTENSION).toString());
				} catch (Exception e) {
					LOGGER.warn("Cannot open template " + template);
					throw new Exception("Cannot open " + new StringBuilder().append(INVOICE_TEMPLATE).append(INVOICE_TEMPLATE_EXTENSION).toString());
				}
			}
			
			if(is==null) {
				LOGGER.warn("Cannot open template " + template);
				throw new Exception("Cannot open " + new StringBuilder().append(INVOICE_TEMPLATE).append(INVOICE_TEMPLATE_EXTENSION).toString());
			}
			
			File file = new File(order.getId() + "_working");
			OutputStream os = new FileOutputStream(file);
			IOUtils.copy(is, os);
			os.close();
			//File file = new File(resource.toURI().toURL());
		
			Sheet sheet = SpreadSheet.createFromFile(file).getSheet(0);
			
			
			//Store name 
			sheet.setValueAt(store.getStorename(), 0, 0);
			
			
			
			//Address
			//count store address cell
			int storeAddressCell = ADDRESS_ROW_START;
			
			Map<String,Zone> zns = zoneService.getZones(language);

			
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
				
				for(Zone z : zones) {
					if(z.getCode().equals(store.getZone().getCode())) {
						storeProvince.append(z.getName());
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
			for(int i = storeAddressCell; i<ADDRESS_ROW_END; i++) {
				sheet.setValueAt("", 0, i);
			}

			//invoice date
			SimpleDateFormat format = new SimpleDateFormat(Constants.DEFAULT_DATE_FORMAT);
			sheet.setValueAt(format.format(order.getDatePurchased()), 3, 2);
			
			//invoice number
			sheet.setValueAt(order.getId(), 3, 3);
			
			//bill to
			//count bill to address cell
			int billToCell = BILLTO_ROW_START;
			if(!StringUtils.isBlank(order.getBilling().getFirstName())) {
				StringBuilder nm = new StringBuilder();
				nm.append(order.getBilling().getFirstName()).append(" ").append(order.getBilling().getLastName());
				sheet.setValueAt(nm.toString(), 0, billToCell);
				billToCell ++;
			}
			
			//9
			if(!StringUtils.isBlank(order.getBilling().getCompany())) {
				sheet.setValueAt(order.getBilling().getCompany(), 0, billToCell);
				billToCell ++;
			}
			
			//10
			StringBuilder billToAddress = null;
			if(!StringUtils.isBlank(order.getBilling().getAddress())) {
				billToAddress = new StringBuilder();
				billToAddress.append(order.getBilling().getAddress());
			}
			if(!StringUtils.isBlank(order.getBilling().getCity())) {
				if(billToAddress==null) {
					billToAddress = new StringBuilder();
				} else {
					billToAddress.append(", ");
				}
				billToAddress.append(order.getBilling().getCity());
			}
			if(billToAddress!=null) {
				sheet.setValueAt(billToAddress.toString(), 0, billToCell);
				billToCell ++;
			}
			
			//11
			StringBuilder billToProvince = null;
			if(order.getBilling().getZone()!=null) {
				billToProvince = new StringBuilder();
				
				Zone billingZone = zns.get(order.getBilling().getZone().getCode());
				if(billingZone!=null) {
						billToProvince.append(billingZone.getName());
				}
				
			} else {
				if(!StringUtils.isBlank(order.getBilling().getState())) {
					billToProvince = new StringBuilder();
					billToProvince.append(order.getBilling().getState());
				}
			}
			if(order.getBilling().getCountry()!=null) {
				if(billToProvince==null) {
					billToProvince = new StringBuilder();
				} else {
					billToProvince.append(", ");
				}
				Country c = countries.get(order.getBilling().getCountry().getIsoCode());
				if(c!=null) {
					billToProvince.append(c.getName());
				} else {
					billToProvince.append(order.getBilling().getCountry().getIsoCode());
				}
				
			}
			if(billToProvince!=null) {
				sheet.setValueAt(billToProvince.toString(), 0, billToCell);
				billToCell ++;
			}
			
			//12
			if(!StringUtils.isBlank(order.getBilling().getPostalCode())) {
				billToCell ++;
				sheet.setValueAt(order.getBilling().getPostalCode(), 0, billToCell);
				billToCell ++;
			}
			
			//13
			if(!StringUtils.isBlank(order.getBilling().getTelephone())) {
				sheet.setValueAt(order.getBilling().getTelephone(), 0, billToCell);
			}
			
			//delete address blank lines
			for(int i = billToCell; i<BILLTO_ROW_END; i++) {
				sheet.setValueAt("", 0, i);
			}
			
			//products
			Set<OrderProduct> orderProducts = order.getOrderProducts();
			int productCell = PRODUCT_ROW_START;
			for(OrderProduct orderProduct : orderProducts) {
				

				String orderProductName = ProductUtils.buildOrderProductDisplayName(orderProduct);
				
				sheet.setValueAt(orderProductName.toString(), 0, productCell);
				
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
				if(totalName.contains(".")) {
					totalName = orderTotal.getTitle();
				}
				String totalValue = priceUtil.getStoreFormatedAmountWithCurrency(store,orderTotal.getValue());
				sheet.setValueAt(totalName, 2, productCell);
				sheet.setValueAt(totalValue, 3, productCell);
				productCell++;
			}
			
			//sheet.getCellAt(0, 0).setImage(arg0)
			//sheet.getCellAt(0, 0).setStyleName(arg0)
			//sheet.getCellAt(0, 0).getStyle().
			
			
			//generate invoice file
			StringBuilder tempInvoiceName = new StringBuilder();
			tempInvoiceName.append(order.getId()).append(TEMP_INVOICE_SUFFIX_NAME);
			File outputFile = new File(tempInvoiceName.toString());
			OOUtils.open(sheet.getSpreadSheet().saveAs(outputFile));
			
			
			
			final OpenDocument doc = new OpenDocument();
			doc.loadFrom(tempInvoiceName.toString());

			 // Open the PDF document
			 Document document = new Document(PageSize.A4);
			 
			 
			 //File outFile = new File("invoice.pdf");

			 PdfDocument pdf = new PdfDocument();

			 document.addDocListener(pdf);

			 //FileOutputStream fileOutputStream = new FileOutputStream(outFile);
			 ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

			 
			 PdfWriter writer = PdfWriter.getInstance(pdf, outputStream);
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
			 float offsetX = (float) ((pageSize.getWidth() - w) / 2);
			 float offsetY = (float) ((pageSize.getHeight() - h) / 2);
			 cb.addTemplate(tp, offsetX, offsetY);
			 // Close the PDF document
			 document.close();
			 outputFile.delete();//remove temp file
			 file.delete();//remove spreadsheet file
			 is.close();
			 return outputStream;
		
		

	}
*/
}



```
