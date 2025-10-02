# EmailTemplatesUtils.java

## Review

## 1. Summary  

`EmailTemplatesUtils` is a Spring `@Component` that centralises the construction and dispatch of all transactional e‑mails in the Shopizer shop.  
The class:

| Responsibility | What it does |
|-----------------|--------------|
| **Email composition** | Builds HTML fragments (addresses, order tables, download links…) and maps them to token placeholders defined in `EmailConstants`. |
| **Email dispatch** | Uses `EmailService.sendHtmlEmail` (which is asynchronous thanks to Spring’s `@Async`) to send the final e‑mail. |
| **Dependency injection** | Pulls in a variety of domain services (pricing, country, zone, product, etc.) and helper utilities (`LabelUtils`, `EmailUtils`, `FilePathUtils`). |

The class is heavily focussed on **business‑logic** – it knows how to format an order, how to pull localized messages, and how to assemble the correct payload for the e‑mail template engine.  
No external framework (apart from Spring, SLF4J, and a handful of Shopizer‑specific services) is involved.  

The design is straightforward: a single component with a method per e‑mail type.  
There is no use of the Strategy or Builder pattern – the logic for building the token map is repeated verbatim across the methods.  

---

## 2. Detailed Description  

### 2.1 Execution Flow (per method)

1. **Entry** – the method is called from various service layers (e.g. after an order is created, or after a user registers).  
2. **Build Token Map** – `emailUtils.createEmailObjectsMap(...)` is called first, producing a baseline map (store URL, locale, etc.).  
3. **Augment Map** – each method adds a handful of tokens that are specific to the e‑mail type (e.g. order ID, product table, status).  
4. **Email Object** – an `Email` POJO is created, populated with `from`, `to`, `subject`, `templateName`, and the token map.  
5. **Send** – `emailService.sendHtmlEmail(...)` is invoked. The method is annotated with `@Async`, so the call is returned immediately and the actual dispatch happens on a Spring executor thread.  
6. **Exception Handling** – any exception thrown during the construction or sending process is caught and logged; the method never propagates the exception back to the caller.  

### 2.2 Key Components  

| Component | Role |
|-----------|------|
| `EmailService` | Thin façade over an e‑mail provider (SMTP, SES, etc.). |
| `EmailUtils` | Builds a *generic* token map (store URL, locale strings, etc.). |
| `LabelUtils` | Internationalisation helper – fetches translated messages. |
| `CountryService` / `ZoneService` | Resolve ISO codes to full names. |
| `PricingService` | Formats monetary amounts for the store’s currency. |
| `FilePathUtils` | Constructs relative URLs for the store, customer portal, etc. |

### 2.3 Architecture & Design Choices  

* **Centralised e‑mail builder** – All e‑mail types share the same component, ensuring consistent formatting and a single place to modify the template logic.  
* **Asynchronous dispatch** – `@Async` offloads e‑mail sending to a thread pool, reducing request latency.  
* **Hard‑coded HTML fragments** – The table structures are written directly in Java (via string constants). This keeps the code self‑contained but makes future HTML changes tedious.  
* **Repetition** – The logic for building the token map is duplicated in every method. A builder or a helper method could eliminate this duplication.  

### 2.4 Assumptions & Constraints  

* The incoming `Order` object is fully initialised (`billing`, `delivery`, `orderProducts`, etc.).  
* All locale‑dependent strings are available in the `messages` bundle.  
* The e‑mail templates referenced by `EmailConstants` contain the tokens defined in the code.  
* The `EmailService` handles its own retry and error handling; this component merely logs failures.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `sendOrderEmail(...)` | Sends an order‑confirmation e‑mail after payment is completed. | `toEmail`, `Customer`, `Order`, `Locale`, `Language`, `MerchantStore`, `contextPath` | None (void) | Builds billing/shipping tables, populates token map, sends e‑mail. |
| `sendRegistrationEmail(...)` | Sends a welcome e‑mail to a newly registered customer. | `PersistableCustomer`, `MerchantStore`, `Locale`, `contextPath` | None | Adds customer credentials to token map, sends e‑mail. |
| `sendContactEmail(...)` | Sends a contact‑form submission to the store owner. | `ContactForm`, `MerchantStore`, `Locale`, `contextPath` | None | Populates contact fields, sends e‑mail. |
| `sendUpdateOrderStatusEmail(...)` | Sends an e‑mail informing the customer of a status change. | `Customer`, `Order`, `OrderStatusHistory`, `MerchantStore`, `Locale`, `contextPath` | None | Builds status message, sends e‑mail. |
| `sendOrderDownloadEmail(...)` | Sends instructions for downloading order‑related content. | `Customer`, `Order`, `MerchantStore`, `Locale`, `contextPath` | None | Builds download message, sends e‑mail. |
| `changePasswordNotificationEmail(...)` | Sends a notification that a customer’s password has changed. | `Customer`, `MerchantStore`, `Locale`, `contextPath` | None | Builds notification message, sends e‑mail. |

All methods are `@Async`, so their execution is non‑blocking and the caller receives a `Future`/`void` immediately.

---

## 4. Dependencies  

| Library / Framework | Type | Notes |
|----------------------|------|-------|
| **Spring Framework** | Standard | Provides `@Component`, `@Async`, `@Inject`, `@Qualifier`, `@Qualifier`. |
| **SLF4J** | Logging | Used for debug and error logs. |
| **Apache Commons Lang3** (`StringUtils`) | Utility | For string checks. |
| **Shopizer core services** | Domain | `EmailService`, `PricingService`, `CountryService`, `ZoneService`, `ProductService`, etc. |
| **Shopizer utils** | Helper | `LabelUtils`, `EmailUtils`, `FilePathUtils`, `ImageFilePath` (unused). |
| **Java SE** | Standard | `Locale`, `Date`, collections. |

All dependencies are either Spring / Java SE or Shopizer‑specific. No external HTTP clients or database access are invoked directly here.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Potential Bugs  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **`shipping` may be `null`** – if `order.getDelivery()` is null and `order.getShippingModuleCode()` is blank, `shipping` remains null and `shipping.toString()` throws `NullPointerException`. | Failure to send e‑mail. | Add a guard: `if (shipping == null) shipping = new StringBuilder();` or simply skip the shipping block when absent. |
| **Unvalidated input** – e.g., `toEmail` may be null or malformed. | `EmailService` could throw or silently fail. | Validate e‑mail format before sending. |
| **Duplicate code** – token map construction is repeated in every method. | Hard to maintain, more room for inconsistencies. | Create a private helper (`buildOrderEmailTokens(...)`) or use a builder pattern. |
| **Hard‑coded strings** – The “welcome” log in `sendOrderEmail` and the `EMAIL_CUSTOMER_CONTACT` token in `sendContactEmail` are misleading or unused. | Documentation confusion. | Correct log messages and remove unused tokens. |
| **`ImageFilePath imageUtils` is injected but never used** – unnecessary injection. | Minor memory/bean‑instantiation overhead. | Remove the injection or integrate image handling if needed. |
| **Potential HTML invalidity** – The table for order totals adds an empty `<td>` before the label and value cells. | Some clients may render incorrectly, but browsers are forgiving. | Confirm the template expects this layout or refactor to a cleaner structure. |
| **No retry policy** – Exceptions are swallowed after logging. | Lost e‑mail delivery. | Either let the exception propagate to a caller that can retry or integrate a retry wrapper around `EmailService`. |
| **Sensitive data** – `sendContactEmail` uses `contact.getName()` as the *from* name. | Possible injection of malicious HTML or header injection. | Sanitize or escape the name before putting it into the e‑mail body. |

### 5.2 Performance & Concurrency  

* The class is already async, but the default executor may be a single‑threaded pool unless explicitly configured.  
  * **Impact** – High e‑mail traffic could block.  
  * **Fix** – Define a dedicated `@Async` executor with a sensible thread‑pool size (`TaskExecutor` bean).  

* The use of `CountryService` and `ZoneService` inside every `sendOrderEmail` call can be expensive if the same order is processed in a short time frame.  
  * **Fix** – Cache the results per request or at the service level.  

### 5.3 Maintainability  

* **HTML fragments** are embedded in Java. Any future design change (e.g., switching from `<table>` to CSS‑grid) would require a full code‑review.  
  * **Recommendation** – Use a templating engine (FreeMarker/Thymeleaf) and keep the HTML in separate files. The token map logic can remain in Java, but the HTML becomes easier to maintain.  

* **Hard‑coded message keys** (`new StringBuilder().append("payment.type.")...`) are brittle.  
  * **Recommendation** – Use `String.format("payment.type.%s", order.getPaymentType().name())` or a dedicated method to build the key.

### 5.4 Testing

* **Unit tests** are currently absent.  
  * **Suggestion** – Mock all injected services (`EmailService`, `PricingService`, etc.) and verify that the token map contains the expected keys and values.  
* **Integration tests** – Use a `MockEmailService` that records sent e‑mails instead of actually dispatching them.  

### 5.5 Documentation & Code‑Quality  

* The class-level comment and in‑method comments are concise but often misleading (`Sending welcome email` in an order‑confirmation method).  
* A **TODO** for image handling indicates incomplete functionality; either remove the unused injection or implement the intended feature.  

---

### 5.6 Suggested Refactor Outline  

```java
public class EmailTemplatesUtils {

    // ---------- shared logic ----------
    private void populateBasicTokens(Map<String, String> tokens, 
                                     MerchantStore store, 
                                     Locale locale,
                                     String contextPath) {
        tokens.putAll(emailUtils.createEmailObjectsMap(contextPath, store, messages, locale));
        tokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", locale));
        // ... any other common tokens
    }

    private String buildAddressTable(Address addr, String title) { … }

    private String buildOrderTable(Order order) { … }

    // ---------- individual e‑mail methods ----------
    public void sendOrderEmail(...) {
        Map<String, String> tokens = new HashMap<>();
        populateBasicTokens(tokens, store, locale, contextPath);
        // add order‑specific tokens
        …
    }
}
```

This removes duplicate code, makes the class easier to read, and simplifies future modifications.

---

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import com.salesmanager.core.business.modules.email.Email;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.zone.ZoneService;
import com.salesmanager.core.business.services.system.EmailService;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.order.OrderTotal;
import com.salesmanager.core.model.order.orderproduct.OrderProduct;
import com.salesmanager.core.model.order.orderstatus.OrderStatusHistory;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.shop.constants.ApplicationConstants;
import com.salesmanager.shop.constants.EmailConstants;
import com.salesmanager.shop.model.customer.PersistableCustomer;
import com.salesmanager.shop.model.shop.ContactForm;
import org.apache.commons.lang3.StringUtils;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

import javax.inject.Inject;
import java.util.Date;
import java.util.Locale;
import java.util.Map;


@Component
public class EmailTemplatesUtils {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(EmailTemplatesUtils.class);
	
	@Inject
	private EmailService emailService;

	@Inject
	private LabelUtils messages;
	
	@Inject
	private CountryService countryService;
	
	@Inject
	private ProductService productService;
	
	@Inject
	private ZoneService zoneService;
	
	@Inject
	private PricingService pricingService;
	
	@Inject
	@Qualifier("img")
	private ImageFilePath imageUtils;
	
	@Inject
	private EmailUtils emailUtils;
	
	@Inject
	private FilePathUtils filePathUtils;
	
	private final static String LINE_BREAK = "<br/>";
	private final static String TABLE = "<table width=\"100%\">";
	private final static String CLOSING_TABLE = "</table>";
	private final static String TR = "<tr>";
	private final static String TR_BORDER = "<tr class=\"border\">";
	private final static String CLOSING_TR = "</tr>";
	private final static String TD = "<td valign=\"top\">";
	private final static String CLOSING_TD = "</td>";
	

	/**
	 * Sends an email to the customer after a completed order
	 * @param customer
	 * @param order
	 * @param customerLocale
	 * @param language
	 * @param merchantStore
	 * @param contextPath
	 */
	@Async
	public void sendOrderEmail(String toEmail, Customer customer, Order order, Locale customerLocale, Language language, MerchantStore merchantStore, String contextPath) {
			   /** issue with putting that elsewhere **/ 
		       LOGGER.info( "Sending welcome email to customer" );
		       try {
		    	   
		    	   Map<String,Zone> zones = zoneService.getZones(language);
		    	   
		    	   Map<String,Country> countries = countryService.getCountriesMap(language);
		    	   
		    	   //format Billing address
		    	   StringBuilder billing = new StringBuilder();
		    	   if(StringUtils.isBlank(order.getBilling().getCompany())) {
		    		   billing.append(order.getBilling().getFirstName()).append(" ")
		    		   .append(order.getBilling().getLastName()).append(LINE_BREAK);
		    	   } else {
		    		   billing.append(order.getBilling().getCompany()).append(LINE_BREAK);
		    	   }
		    	   billing.append(order.getBilling().getAddress()).append(LINE_BREAK);
		    	   billing.append(order.getBilling().getCity()).append(", ");
		    	   
		    	   if(order.getBilling().getZone()!=null) {
		    		   Zone zone = zones.get(order.getBilling().getZone().getCode());
		    		   if(zone!=null) {
		    			   billing.append(zone.getName());
		    		   }
		    		   billing.append(LINE_BREAK);
		    	   } else if(!StringUtils.isBlank(order.getBilling().getState())) {
		    		   billing.append(order.getBilling().getState()).append(LINE_BREAK); 
		    	   }
		    	   Country country = countries.get(order.getBilling().getCountry().getIsoCode());
		    	   if(country!=null) {
		    		   billing.append(country.getName()).append(" ");
		    	   }
		    	   billing.append(order.getBilling().getPostalCode());
		    	   
		    	   
		    	   //format shipping address
		    	   StringBuilder shipping = null;
		    	   if(order.getDelivery()!=null && !StringUtils.isBlank(order.getDelivery().getFirstName())) {
		    		   shipping = new StringBuilder();
			    	   if(StringUtils.isBlank(order.getDelivery().getCompany())) {
			    		   shipping.append(order.getDelivery().getFirstName()).append(" ")
			    		   .append(order.getDelivery().getLastName()).append(LINE_BREAK);
			    	   } else {
			    		   shipping.append(order.getDelivery().getCompany()).append(LINE_BREAK);
			    	   }
			    	   shipping.append(order.getDelivery().getAddress()).append(LINE_BREAK);
			    	   shipping.append(order.getDelivery().getCity()).append(", ");
			    	   
			    	   if(order.getDelivery().getZone()!=null) {
			    		   Zone zone = zones.get(order.getDelivery().getZone().getCode());
			    		   if(zone!=null) {
			    			   shipping.append(zone.getName());
			    		   }
			    		   shipping.append(LINE_BREAK);
			    	   } else if(!StringUtils.isBlank(order.getDelivery().getState())) {
			    		   shipping.append(order.getDelivery().getState()).append(LINE_BREAK); 
			    	   }
			    	   Country deliveryCountry = countries.get(order.getDelivery().getCountry().getIsoCode());
			    	   if(country!=null) {
			    		   shipping.append(deliveryCountry.getName()).append(" ");
			    	   }
			    	   shipping.append(order.getDelivery().getPostalCode());
		    	   }
		    	   
		    	   if(shipping==null && StringUtils.isNotBlank(order.getShippingModuleCode())) {
		    		   //TODO IF HAS NO SHIPPING
		    		   shipping = billing;
		    	   }
		    	   
		    	   //format order
		    	   //String storeUri = FilePathUtils.buildStoreUri(merchantStore, contextPath);
		    	   StringBuilder orderTable = new StringBuilder();
		    	   orderTable.append(TABLE);
		    	   for(OrderProduct product : order.getOrderProducts()) {
		    		   //Product productModel = productService.getByCode(product.getSku(), language);
		    		   orderTable.append(TR);
			    		   orderTable.append(TD).append(product.getProductName()).append(" - ").append(product.getSku()).append(CLOSING_TD);
		    		   	   orderTable.append(TD).append(messages.getMessage("label.quantity", customerLocale)).append(": ").append(product.getProductQuantity()).append(CLOSING_TD);
	    		   		   orderTable.append(TD).append(pricingService.getDisplayAmount(product.getOneTimeCharge(), merchantStore)).append(CLOSING_TD);
    		   		   orderTable.append(CLOSING_TR);
		    	   }

		    	   //order totals
		    	   for(OrderTotal total : order.getOrderTotal()) {
		    		   orderTable.append(TR_BORDER);
		    		   		//orderTable.append(TD);
		    		   		//orderTable.append(CLOSING_TD);
		    		   		orderTable.append(TD);
		    		   		orderTable.append(CLOSING_TD);
		    		   		orderTable.append(TD);
		    		   		orderTable.append("<strong>");
		    		   			if(total.getModule().equals("tax")) {
		    		   				orderTable.append(total.getText()).append(": ");

		    		   			} else {
		    		   				//if(total.getModule().equals("total") || total.getModule().equals("subtotal")) {
		    		   				//}
		    		   				orderTable.append(messages.getMessage(total.getOrderTotalCode(), customerLocale)).append(": ");
		    		   				//if(total.getModule().equals("total") || total.getModule().equals("subtotal")) {
		    		   					
		    		   				//}
		    		   			}
		    		   		orderTable.append("</strong>");
		    		   		orderTable.append(CLOSING_TD);
		    		   		orderTable.append(TD);
		    		   			orderTable.append("<strong>");

		    		   			orderTable.append(pricingService.getDisplayAmount(total.getValue(), merchantStore));

	    		   				orderTable.append("</strong>");
		    		   		orderTable.append(CLOSING_TD);
		    		   orderTable.append(CLOSING_TR);
		    	   }
		    	   orderTable.append(CLOSING_TABLE);

		           Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, customerLocale);
		           templateTokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", customerLocale));
		           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_FIRSTNAME, order.getBilling().getFirstName());
		           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_LASTNAME, order.getBilling().getLastName());
		           
		           String[] params = {String.valueOf(order.getId())};
		           String[] dt = {DateUtil.formatDate(order.getDatePurchased())};
		           templateTokens.put(EmailConstants.EMAIL_ORDER_NUMBER, messages.getMessage("email.order.confirmation", params, customerLocale));
		           templateTokens.put(EmailConstants.EMAIL_ORDER_DATE, messages.getMessage("email.order.ordered", dt, customerLocale));
		           templateTokens.put(EmailConstants.EMAIL_ORDER_THANKS, messages.getMessage("email.order.thanks",customerLocale));
		           templateTokens.put(EmailConstants.ADDRESS_BILLING, billing.toString());
		           
		           templateTokens.put(EmailConstants.ORDER_PRODUCTS_DETAILS, orderTable.toString());
		           templateTokens.put(EmailConstants.EMAIL_ORDER_DETAILS_TITLE, messages.getMessage("label.order.details",customerLocale));
		           templateTokens.put(EmailConstants.ADDRESS_BILLING_TITLE, messages.getMessage("label.customer.billinginformation",customerLocale));
		           templateTokens.put(EmailConstants.PAYMENT_METHOD_TITLE, messages.getMessage("label.order.paymentmode",customerLocale));
		           templateTokens.put(EmailConstants.PAYMENT_METHOD_DETAILS, messages.getMessage(new StringBuilder().append("payment.type.").append(order.getPaymentType().name()).toString(),customerLocale,order.getPaymentType().name()));
		           
		           if(StringUtils.isNotBlank(order.getShippingModuleCode())) {
		        	   //templateTokens.put(EmailConstants.SHIPPING_METHOD_DETAILS, messages.getMessage(new StringBuilder().append("module.shipping.").append(order.getShippingModuleCode()).toString(),customerLocale,order.getShippingModuleCode()));
		        	   templateTokens.put(EmailConstants.SHIPPING_METHOD_DETAILS, messages.getMessage(new StringBuilder().append("module.shipping.").append(order.getShippingModuleCode()).toString(),new String[]{merchantStore.getStorename()},customerLocale));
		        	   templateTokens.put(EmailConstants.ADDRESS_SHIPPING_TITLE, messages.getMessage("label.order.shippingmethod",customerLocale));
		        	   templateTokens.put(EmailConstants.ADDRESS_DELIVERY_TITLE, messages.getMessage("label.customer.shippinginformation",customerLocale));
		        	   templateTokens.put(EmailConstants.SHIPPING_METHOD_TITLE, messages.getMessage("label.customer.shippinginformation",customerLocale));
		        	   templateTokens.put(EmailConstants.ADDRESS_DELIVERY, shipping.toString());
		           } else {
		        	   templateTokens.put(EmailConstants.SHIPPING_METHOD_DETAILS, "");
		        	   templateTokens.put(EmailConstants.ADDRESS_SHIPPING_TITLE, "");
		        	   templateTokens.put(EmailConstants.ADDRESS_DELIVERY_TITLE, "");
		        	   templateTokens.put(EmailConstants.SHIPPING_METHOD_TITLE, "");
		        	   templateTokens.put(EmailConstants.ADDRESS_DELIVERY, "");
		           }
		           
			       String status = messages.getMessage("label.order." + order.getStatus().name(), customerLocale, order.getStatus().name());
			       String[] statusMessage = {DateUtil.formatDate(order.getDatePurchased()),status};
		           templateTokens.put(EmailConstants.ORDER_STATUS, messages.getMessage("email.order.status", statusMessage, customerLocale));
		           

		           String[] title = {merchantStore.getStorename(), String.valueOf(order.getId())};
		           Email email = new Email();
		           email.setFrom(merchantStore.getStorename());
		           email.setFromEmail(merchantStore.getStoreEmailAddress());
		           email.setSubject(messages.getMessage("email.order.title", title, customerLocale));
		           email.setTo(toEmail);
		           email.setTemplateName(EmailConstants.EMAIL_ORDER_TPL);
		           email.setTemplateTokens(templateTokens);

		           LOGGER.debug( "Sending email to {} for order id {} ",customer.getEmailAddress(), order.getId() );
		           emailService.sendHtmlEmail(merchantStore, email);

		       } catch (Exception e) {
		           LOGGER.error("Error occured while sending order confirmation email ",e);
		       }
			
		}
	
	/**
	 * Sends an email to the customer after registration
	 * @param request
	 * @param customer
	 * @param merchantStore
	 * @param customerLocale
	 */
	@Async
	public void sendRegistrationEmail(
		PersistableCustomer customer, MerchantStore merchantStore,
			Locale customerLocale, String contextPath) {
		   /** issue with putting that elsewhere **/ 
	       LOGGER.info( "Sending welcome email to customer" );
	       try {

	           Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, customerLocale);
	           templateTokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", customerLocale));
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_FIRSTNAME, customer.getBilling().getFirstName());
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_LASTNAME, customer.getBilling().getLastName());
	           String[] greetingMessage = {merchantStore.getStorename(),filePathUtils.buildCustomerUri(merchantStore,contextPath),merchantStore.getStoreEmailAddress()};
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_GREETING, messages.getMessage("email.customer.greeting", greetingMessage, customerLocale));
	           templateTokens.put(EmailConstants.EMAIL_USERNAME_LABEL, messages.getMessage("label.generic.username",customerLocale));
	           templateTokens.put(EmailConstants.EMAIL_PASSWORD_LABEL, messages.getMessage("label.generic.password",customerLocale));
	           templateTokens.put(EmailConstants.CUSTOMER_ACCESS_LABEL, messages.getMessage("label.customer.accessportal",customerLocale));
	           templateTokens.put(EmailConstants.ACCESS_NOW_LABEL, messages.getMessage("label.customer.accessnow",customerLocale));
	           templateTokens.put(EmailConstants.EMAIL_USER_NAME, customer.getUserName());
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_PASSWORD, customer.getPassword());

	           //shop url
	           String customerUrl = filePathUtils.buildStoreUri(merchantStore, contextPath);
	           templateTokens.put(EmailConstants.CUSTOMER_ACCESS_URL, customerUrl);

	           Email email = new Email();
	           email.setFrom(merchantStore.getStorename());
	           email.setFromEmail(merchantStore.getStoreEmailAddress());
	           email.setSubject(messages.getMessage("email.newuser.title",customerLocale));
	           email.setTo(customer.getEmailAddress());
	           email.setTemplateName(EmailConstants.EMAIL_CUSTOMER_TPL);
	           email.setTemplateTokens(templateTokens);

	           LOGGER.debug( "Sending email to {} on their  registered email id {} ",customer.getBilling().getFirstName(),customer.getEmailAddress() );
	           emailService.sendHtmlEmail(merchantStore, email);

	       } catch (Exception e) {
	           LOGGER.error("Error occured while sending welcome email ",e);
	       }
		
	}
	
	@Async
	public void sendContactEmail(
			ContactForm contact, MerchantStore merchantStore,
				Locale storeLocale, String contextPath) {
			   /** issue with putting that elsewhere **/ 
		       LOGGER.info( "Sending email to store owner" );
		       try {

		           Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, storeLocale);
		           
		           templateTokens.put(EmailConstants.EMAIL_CONTACT_NAME, contact.getName());
		           templateTokens.put(EmailConstants.EMAIL_CONTACT_EMAIL, contact.getEmail());
		           templateTokens.put(EmailConstants.EMAIL_CONTACT_CONTENT, contact.getComment());
		           
		           String[] contactSubject = {contact.getSubject()};
		           
		           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_CONTACT, messages.getMessage("email.contact",contactSubject, storeLocale));
		           templateTokens.put(EmailConstants.EMAIL_CONTACT_NAME_LABEL, messages.getMessage("label.entity.name",storeLocale));
		           templateTokens.put(EmailConstants.EMAIL_CONTACT_EMAIL_LABEL, messages.getMessage("label.generic.email",storeLocale));



		           Email email = new Email();
		           email.setFrom(contact.getName());
		           //since shopizer sends email to store email, sender is store email
		           email.setFromEmail(merchantStore.getStoreEmailAddress());
		           email.setSubject(messages.getMessage("email.contact.title",storeLocale));
		           //contact has to be delivered to store owner, receiver is store email
		           email.setTo(merchantStore.getStoreEmailAddress());
		           email.setTemplateName(EmailConstants.EMAIL_CONTACT_TMPL);
		           email.setTemplateTokens(templateTokens);

		           LOGGER.debug( "Sending contact email");
		           emailService.sendHtmlEmail(merchantStore, email);

		       } catch (Exception e) {
		           LOGGER.error("Error occured while sending contact email ",e);
		       }
			
		}
	
	/**
	 * Send an email to the customer with last order status
	 * @param request
	 * @param customer
	 * @param order
	 * @param merchantStore
	 * @param customerLocale
	 */
	@Async
	public void sendUpdateOrderStatusEmail(
			Customer customer, Order order, OrderStatusHistory lastHistory, MerchantStore merchantStore,
			Locale customerLocale, String contextPath) {
		   /** issue with putting that elsewhere **/ 
	       LOGGER.info( "Sending order status email to customer" );
	       try {


				Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, customerLocale);
				
		        templateTokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", customerLocale));
		        templateTokens.put(EmailConstants.EMAIL_CUSTOMER_FIRSTNAME, customer.getBilling().getFirstName());
		        templateTokens.put(EmailConstants.EMAIL_CUSTOMER_LASTNAME, customer.getBilling().getLastName());
				
		        String[] statusMessageText = {String.valueOf(order.getId()),DateUtil.formatDate(order.getDatePurchased())};
		        String status = messages.getMessage("label.order." + order.getStatus().name(), customerLocale, order.getStatus().name());
		        String[] statusMessage = {DateUtil.formatDate(lastHistory.getDateAdded()),status};
		        
		        String comments = lastHistory.getComments();
		        if(StringUtils.isBlank(comments)) {
		        	comments = messages.getMessage("label.order." + order.getStatus().name(), customerLocale, order.getStatus().name());
		        }
		        
				templateTokens.put(EmailConstants.EMAIL_ORDER_STATUS_TEXT, messages.getMessage("email.order.statustext", statusMessageText, customerLocale));
				templateTokens.put(EmailConstants.EMAIL_ORDER_STATUS, messages.getMessage("email.order.status", statusMessage, customerLocale));
				templateTokens.put(EmailConstants.EMAIL_TEXT_STATUS_COMMENTS, comments);
				
				
				Email email = new Email();
				email.setFrom(merchantStore.getStorename());
				email.setFromEmail(merchantStore.getStoreEmailAddress());
				email.setSubject(messages.getMessage("email.order.status.title",new String[]{String.valueOf(order.getId())},customerLocale));
				email.setTo(customer.getEmailAddress());
				email.setTemplateName(EmailConstants.ORDER_STATUS_TMPL);
				email.setTemplateTokens(templateTokens);
	
	
				
				emailService.sendHtmlEmail(merchantStore, email);

	       } catch (Exception e) {
	           LOGGER.error("Error occured while sending order download email ",e);
	       }
		
	}
	
	/**
	 * Send download email instructions to customer
	 * @param customer
	 * @param order
	 * @param merchantStore
	 * @param customerLocale
	 * @param contextPath
	 */
	@Async
	public void sendOrderDownloadEmail(
			Customer customer, Order order, MerchantStore merchantStore,
			Locale customerLocale, String contextPath) {
		   /** issue with putting that elsewhere **/ 
	       LOGGER.info( "Sending download email to customer" );
	       try {

	           Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, customerLocale);
	           templateTokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", customerLocale));
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_FIRSTNAME, customer.getBilling().getFirstName());
	           templateTokens.put(EmailConstants.EMAIL_CUSTOMER_LASTNAME, customer.getBilling().getLastName());
	           String[] downloadMessage = {String.valueOf(ApplicationConstants.MAX_DOWNLOAD_DAYS), String.valueOf(order.getId()), filePathUtils.buildCustomerUri(merchantStore, contextPath), merchantStore.getStoreEmailAddress()};
	           templateTokens.put(EmailConstants.EMAIL_ORDER_DOWNLOAD, messages.getMessage("email.order.download.text", downloadMessage, customerLocale));
	           templateTokens.put(EmailConstants.CUSTOMER_ACCESS_LABEL, messages.getMessage("label.customer.accessportal",customerLocale));
	           templateTokens.put(EmailConstants.ACCESS_NOW_LABEL, messages.getMessage("label.customer.accessnow",customerLocale));

	           //shop url
	           String customerUrl = filePathUtils.buildStoreUri(merchantStore, contextPath);
	           templateTokens.put(EmailConstants.CUSTOMER_ACCESS_URL, customerUrl);

	           String[] orderInfo = {String.valueOf(order.getId())};
	           
	           Email email = new Email();
	           email.setFrom(merchantStore.getStorename());
	           email.setFromEmail(merchantStore.getStoreEmailAddress());
	           email.setSubject(messages.getMessage("email.order.download.title", orderInfo, customerLocale));
	           email.setTo(customer.getEmailAddress());
	           email.setTemplateName(EmailConstants.EMAIL_ORDER_DOWNLOAD_TPL);
	           email.setTemplateTokens(templateTokens);

	           LOGGER.debug( "Sending email to {} with download info",customer.getEmailAddress() );
	           emailService.sendHtmlEmail(merchantStore, email);

	       } catch (Exception e) {
	           LOGGER.error("Error occured while sending order download email ",e);
	       }
		
	}
	
	/**
	 * Sends a change password notification email to the Customer
	 * @param customer
	 * @param merchantStore
	 * @param customerLocale
	 * @param contextPath
	 */
	@Async
	public void changePasswordNotificationEmail(
			Customer customer, MerchantStore merchantStore,
			Locale customerLocale, String contextPath) {
	       LOGGER.debug( "Sending change password email" );
	       try {


				Map<String, String> templateTokens = emailUtils.createEmailObjectsMap(contextPath, merchantStore, messages, customerLocale);
				
		        templateTokens.put(EmailConstants.LABEL_HI, messages.getMessage("label.generic.hi", customerLocale));
		        templateTokens.put(EmailConstants.EMAIL_CUSTOMER_FIRSTNAME, customer.getBilling().getFirstName());
		        templateTokens.put(EmailConstants.EMAIL_CUSTOMER_LASTNAME, customer.getBilling().getLastName());
				
		        String[] date = {DateUtil.formatLongDate(new Date())};
		        
		        templateTokens.put(EmailConstants.EMAIL_NOTIFICATION_MESSAGE, messages.getMessage("label.notification.message.passwordchanged", date, customerLocale));
		        

				Email email = new Email();
				email.setFrom(merchantStore.getStorename());
				email.setFromEmail(merchantStore.getStoreEmailAddress());
				email.setSubject(messages.getMessage("label.notification.title.passwordchanged",customerLocale));
				email.setTo(customer.getEmailAddress());
				email.setTemplateName(EmailConstants.EMAIL_NOTIFICATION_TMPL);
				email.setTemplateTokens(templateTokens);
	
	
				
				emailService.sendHtmlEmail(merchantStore, email);

	       } catch (Exception e) {
	           LOGGER.error("Error occured while sending change password email ",e);
	       }
		
	}

}


```
