# PaymentServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`PaymentServiceImpl` is a Spring‑managed service that orchestrates all payment‑related operations in the *SalesManager* platform.  It is the single point of contact between the order‑processing workflow and the various external payment providers (e.g. Stripe, PayPal, Authorize.Net).  

**Key responsibilities**

| Responsibility | Component / method |
|----------------|-------------------|
| Discover available payment modules for a store | `getPaymentMethods`, `getAcceptedPaymentMethods` |
| Load and persist encrypted payment configuration | `getPaymentModulesConfigured`, `savePaymentModuleConfiguration`, `removePaymentModuleConfiguration` |
| Process payments (authorize, capture, init) | `processPayment`, `processCapturePayment`, `initTransaction` |
| Process refunds | `processRefund` |
| Validate credit‑card data | `validateCreditCard`, `validateCreditCardDate`, `validateCreditCardNumber`, `luhnValidate` |
| Resolve a payment module instance by code | `getPaymentModule`, `getPaymentMethodByCode` |

**Design patterns & libraries**

* **Strategy** – Each concrete payment provider implements the `PaymentModule` interface.  
* **Decorator / Adapter** – `IntegrationConfiguration` objects are wrapped into `IntegrationModule` instances.  
* **Factory / Dependency Injection** – Spring’s `@Inject` / `@Resource` wires all dependencies; the map of payment modules (`paymentModules`) is injected from the application context.  
* **Encryption** – A simple AES wrapper (`Encryption`) is used to encrypt/decrypt the JSON‑string of configuration values.  
* **Commons Lang** – String and validation utilities (`StringUtils`, `Validate`).  
* **SLF4J** – Logging.

---

## 2. Detailed Description  

### 2.1 Initialization  

On application start, Spring creates a singleton `PaymentServiceImpl`.  All its dependencies are injected:

* `MerchantConfigurationService`, `ModuleConfigurationService`, `TransactionService`, `OrderService`, `CoreConfiguration`, `Encryption`  
* `Map<String, PaymentModule> paymentModules` – a map of all available payment module implementations (keys are the module codes).  

### 2.2 Runtime Flow  

#### 2.2.1 Discovering modules  
`getPaymentMethods` queries `moduleConfigurationService` for all modules labelled *PAYMENT_MODULES*, filters them by the store’s country (or a wildcard `*`), and returns the filtered list.  
`getAcceptedPaymentMethods` looks up the *active* configuration for each module and builds a list of `PaymentMethod` objects that the store is allowed to present to the customer.

#### 2.2.2 Configuration persistence  
Configurations are stored in a single merchant‑specific `MerchantConfiguration` record keyed by `Constants.PAYMENT_MODULES`.  
The record’s `value` is an AES‑encrypted JSON string that contains a map of `IntegrationConfiguration` objects.  
* `getPaymentModulesConfigured` decrypts, parses, and returns the map.  
* `savePaymentModuleConfiguration` validates the module’s own rules (via `module.validateModuleConfiguration`), merges it into the existing map, re‑serialises, encrypts and persists the record.  
* `removePaymentModuleConfiguration` removes a key from the map (or deletes a custom module’s configuration if present).

#### 2.2.3 Payment processing  

`processPayment` is the work‑horse for a full order payment:

1. **Validation** – customer, store, payment, order and totals are non‑null.  
2. **Amount & currency** – currency is set to the store’s default.  
3. **Configuration** – the module’s `IntegrationConfiguration` is fetched and must be active.  
4. **Transaction type** – extracted from the configuration (default *AUTHORIZECAPTURE*).  
5. **Credit‑card validation** – if required, `validateCreditCard` is invoked.  
6. **Module lookup** – the concrete `PaymentModule` is fetched from the map.  
7. **Delegate** – based on the transaction type, the call is forwarded to `module.authorize`, `module.authorizeAndCapture` or `module.initTransaction`.  
8. **Persist transaction** – if not an *INIT* call, the transaction is stored via `transactionService`.  
9. **Order status** – for capture, the order status may be updated to *ORDERED* or *PROCESSED*.  

`processCapturePayment` is similar but works on an existing *capturable* transaction, obtained from `transactionService.getCapturableTransaction(order)`.  
`processRefund` performs a similar flow but works on a *refundable* transaction, updates the order totals, sets status to *REFUNDED*, and logs a status history entry.

#### 2.2.4 Transaction initialisation  
Two overloaded `initTransaction` methods create an *INIT* transaction, which represents a pre‑authorisation or a payment that will be captured later.  The returned `Transaction` is stored with `transactionService.save`.

#### 2.3 Clean‑up  
No explicit clean‑up is required; the service relies on Spring’s bean lifecycle and the underlying DAO layers to manage persistence.

### 2.4 Assumptions & Constraints  

* All payment modules implement the `PaymentModule` interface and are registered in the Spring context.  
* Encrypted configuration JSON is small enough to fit in a single DB field.  
* The `integrationModules` map is static for the application’s lifetime – no dynamic hot‑reload of modules.  
* The `coreConfiguration` bean contains a property `VALIDATE_CREDIT_CARD` that toggles card validation.  
* The code assumes the merchant’s configuration is always present; it creates a new one lazily if absent.  
* No transaction‑level locking or concurrency controls are present around the configuration map – two simultaneous writers could corrupt the JSON.

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `List<IntegrationModule> getPaymentMethods(MerchantStore store)` | Return all payment modules available for a store’s country. | `store` | list of `IntegrationModule` | none |
| `List<PaymentMethod> getAcceptedPaymentMethods(MerchantStore store)` | Return all *active* payment modules that the store is allowed to use. | `store` | list of `PaymentMethod` | none |
| `IntegrationModule getPaymentMethodByType(MerchantStore store, String type)` | Find a module by its type. | `store`, `type` | `IntegrationModule` or `null` | none |
| `IntegrationModule getPaymentMethodByCode(MerchantStore store, String code)` | Find a module by its code. | `store`, `code` | `IntegrationModule` or `null` | none |
| `IntegrationConfiguration getPaymentConfiguration(String moduleCode, MerchantStore store)` | Fetch a single configuration by module code. | `moduleCode`, `store` | `IntegrationConfiguration` or `null` | none |
| `Map<String, IntegrationConfiguration> getPaymentModulesConfigured(MerchantStore store)` | Decrypt and parse the merchant’s payment configuration JSON. | `store` | map of `moduleCode -> IntegrationConfiguration` | none |
| `void savePaymentModuleConfiguration(IntegrationConfiguration configuration, MerchantStore store)` | Persist a configuration key/value pair. | `configuration`, `store` | none | creates/updates merchant configuration, encrypts JSON |
| `void removePaymentModuleConfiguration(String moduleCode, MerchantStore store)` | Delete a configuration entry (or an entire custom module). | `moduleCode`, `store` | none | modifies encrypted JSON or deletes a config entity |
| `Transaction processPayment(Customer, MerchantStore, Payment, List<ShoppingCartItem>, Order)` | Authorise / capture / initialise a payment for an order. | `customer`, `store`, `payment`, `items`, `order` | `Transaction` | persists transaction, updates order status |
| `PaymentModule getPaymentModule(String paymentModuleCode)` | Retrieve the concrete module instance. | `paymentModuleCode` | `PaymentModule` or `null` | none |
| `Transaction processCapturePayment(Order, Customer, MerchantStore)` | Capture a previously authorised transaction. | `order`, `customer`, `store` | `Transaction` | persists transaction, updates order status |
| `Transaction processRefund(Order, Customer, MerchantStore, BigDecimal)` | Refund a payment, possibly partially. | `order`, `customer`, `store`, `amount` | `Transaction` | persists transaction, updates order totals, status |
| `void validateCreditCard(String, CreditCardType, String, String)` | Public wrapper for all card validations. | card number, type, month, year | none | throws `ServiceException` on failure |
| `void validateCreditCardDate(int, int)` | Ensure the expiry date is not in the past. | month, year | none | throws `ServiceException` |
| `void validateCreditCardNumber(String, CreditCardType)` | Validate card number format per card brand. | number, type | none | throws `ServiceException` |
| `void luhnValidate(String)` | Run Luhn algorithm check. | number | none | throws `ServiceException` |
| `Transaction initTransaction(Order, Customer, Payment, MerchantStore)` | Create an *INIT* transaction for an order. | `order`, `customer`, `payment`, `store` | `Transaction` | persists transaction |
| `Transaction initTransaction(Customer, Payment, MerchantStore)` | Create an *INIT* transaction for a standalone payment. | `customer`, `payment`, `store` | `Transaction` | persists transaction |

Reusable utility methods: `validateCreditCard`, `validateCreditCardNumber`, `luhnValidate` – all public/​protected but could be extracted to a dedicated helper class.

---

## 4. Dependencies  

| Library / Framework | Role | Standard / Third‑party |
|----------------------|------|------------------------|
| **Spring Framework** (`@Inject`, `@Resource`, bean wiring) | Dependency injection & bean lifecycle | Third‑party (open‑source) |
| **Apache Commons Lang3** (`StringUtils`, `Validate`) | String handling & argument validation | Third‑party |
| **SLF4J** (with an underlying binding e.g. Logback) | Logging | Third‑party |
| **javax.inject** (`@Inject`) | CDI style injection | Third‑party |
| **java.util** | Standard Java APIs (Collections, BigDecimal, Calendar) | Standard |
| **java.util.regex.Pattern / Matcher** | Regular‑expression validation | Standard |
| **Encryption** | Simple AES wrapper for storing JSON config | Custom code (in the same project) |
| **CoreConfiguration** | Holds runtime properties (e.g. `VALIDATE_CREDIT_CARD`) | Custom |
| **TransactionService, OrderService, etc.** | DAO / persistence layer | Custom |

No direct usage of JPA/Hibernate annotations in this file; the DAO layers are injected and handle the database interactions.

---

## 5. Review & Recommendations  

### 5.1 Strengths  

1. **Clear separation of concerns** – payment logic is decoupled from order persistence, module discovery, and configuration management.  
2. **Security** – payment credentials are never stored in plain text.  
3. **Extensibility** – new payment providers can be added by implementing `PaymentModule` and registering them in the Spring context.  
4. **Rich validation** – the card‑number checks (including Luhn) are thorough.

### 5.2 Weaknesses & Risks  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Concurrent configuration writes** – The JSON string is loaded, modified, re‑encrypted and persisted without any lock.  Two writers can interleave and corrupt the file. | Data loss / incorrect config | Wrap the whole read‑modify‑write in a synchronized block or use optimistic locking (version field) on `MerchantConfiguration`. |
| **Hardcoded card validation** – The `validateCreditCardNumber` and `luhnValidate` methods are marked `@Deprecated` but still used.  They duplicate logic that is already available in *Commons Validator* (`CreditCardValidator`). | Bloat, maintenance burden, potential bugs | Remove custom validation, delegate to `org.apache.commons.validator.routines.CreditCardValidator`. |
| **Large JSON field** – Storing all configs in one field may hit DB limits (e.g. 4000 chars). | DB constraint violations | Split into separate rows (module + config) or use a BLOB. |
| **No transaction isolation** – `processPayment` touches the order, transaction, and configuration in a single flow but does not run them in a database transaction.  Partial failures could leave the system in an inconsistent state. | Inconsistent order status or missing transaction | Wrap the entire method in a `@Transactional` and cascade persistence. |
| **`coreConfiguration` property name** – The code looks for a key `VALIDATE_CREDIT_CARD`.  The property name is not documented anywhere else. | Mis‑configuration leads to silent skip of validation | Document the property in a central config file and provide a typed bean. |
| **Deprecated methods** – `validateCreditCardNumber` and `luhnValidate` are public but marked deprecated.  Future maintainers may overlook the deprecation. | Unclear API evolution | Either remove them or make them `private`/`package‑private` and provide a single entry point. |
| **Redundant `initTransaction(Order…)` overload** – Both overloads call the same internal logic but one uses `order.getPaymentModuleCode()` while the other uses `payment.getModuleName()`.  The semantics are confusing. | API confusion | Keep only one overload that accepts an `Order` (which already contains the module code) or make the other overload a thin wrapper that builds a fake order. |
| **Magic strings** – e.g. `"messages.error.creditcard.number"`.  These are hardcoded, making localisation harder. | Future localisation changes may break. | Use constants or a message‑key service. |
| **Exception handling** – `ServiceException.EXCEPTION_VALIDATION` codes are hard‑coded as strings; better to use enums or a dedicated `ErrorCode` enum. | Inconsistent error reporting | Create an enum `PaymentErrorCode` and map it to the exception. |

### 5.3 Performance & Scalability  

* **Encryption** – AES encryption/decryption per call is negligible, but the JSON parsing is done on every payment request.  Caching the parsed map per merchant would reduce overhead.  
* **Large order items list** – `processPayment` receives a list of `ShoppingCartItem`; however the list is only used when authorising or capturing.  If the module only needs the total, passing a single `BigDecimal` would be lighter.  
* **Module lookup** – Map lookup is O(1), but the service still iterates over all modules for discovery.  If the number of modules grows, consider indexing by country in the DAO layer instead of filtering in memory.

### 5.4 Security  

* All credentials are stored encrypted; the `Encryption` bean must be correctly initialised with a secure key.  
* `validateCreditCard` removes all non‑digit characters before validation, preventing simple injection attacks.  
* Still, the service does not protect against **timing attacks** (the card number checks expose brand‑specific prefixes).  Using a third‑party library (e.g. Apache Commons Validator or Bouncy‑Castle) would provide a hardened implementation.

### 5.5 Recommendations  

| Area | Suggested change |
|------|------------------|
| **Configuration concurrency** | Add a `synchronized` block or use a distributed lock (e.g. Redisson) around the decrypt‑modify‑encrypt sequence. |
| **Card validation** | Remove the custom implementation, rely entirely on `org.apache.commons.validator.routines.CreditCardValidator`.  This will also drop the deprecated methods. |
| **Exception codes** | Create an `enum PaymentErrorCode` and use it throughout instead of hard‑coded strings. |
| **Method naming** | `initTransaction(Order...)` should probably be called `initTransactionForOrder` for clarity. |
| **Logging** | Use parameterised logs (`log.debug("Processing payment of {} for order {}", payment.getAmount(), order.getId());`) to avoid string concatenation when debug is off. |
| **Unit tests** | The service is large; test stubs for each `PaymentModule` and mock the DAOs to assert that correct provider methods are called. |
| **API design** | Expose the payment configuration as a dedicated DTO; return `Optional<IntegrationConfiguration>` instead of `null`. |
| **Error handling** | Wrap all DAO calls in try/catch blocks that translate persistence exceptions into `ServiceException`. |
| **DTO validation** | Use Java Bean Validation (`@NotNull`, `@Digits`, etc.) on the `Payment` and `Order` DTOs so the service can rely on a single `Validator`. |

---

## 5. Overall Assessment  

`PaymentServiceImpl` is a solid, functional implementation that covers the full payment lifecycle while keeping payment‑provider logic encapsulated.  The use of encryption for configuration, the strategy pattern for payment modules, and the integration with the existing *SalesManager* persistence layer are all appropriate for an e‑commerce platform.  

However, the code suffers from a few maintainability and safety issues:

* **Manual credit‑card validation** – duplicates existing libraries and is deprecated, yet still called.  
* **Potential race conditions** when writing the encrypted configuration.  
* **Hard‑coded string literals** for error messages and property names.  
* **Redundant method overloads** and minor API inconsistencies.  

Addressing these points would make the service more robust, easier to test, and future‑proof.  The overall architecture, however, is sound and aligns well with the platform’s modular design philosophy.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.payments;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import javax.annotation.Resource;
import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.order.OrderService;
import com.salesmanager.core.business.services.reference.loader.ConfigurationModulesLoader;
import com.salesmanager.core.business.services.system.MerchantConfigurationService;
import com.salesmanager.core.business.services.system.ModuleConfigurationService;
import com.salesmanager.core.business.utils.CoreConfiguration;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.order.OrderTotal;
import com.salesmanager.core.model.order.OrderTotalType;
import com.salesmanager.core.model.order.orderstatus.OrderStatus;
import com.salesmanager.core.model.order.orderstatus.OrderStatusHistory;
import com.salesmanager.core.model.payments.CreditCardPayment;
import com.salesmanager.core.model.payments.CreditCardType;
import com.salesmanager.core.model.payments.Payment;
import com.salesmanager.core.model.payments.PaymentMethod;
import com.salesmanager.core.model.payments.PaymentType;
import com.salesmanager.core.model.payments.Transaction;
import com.salesmanager.core.model.payments.TransactionType;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.model.system.MerchantConfiguration;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.payment.model.PaymentModule;
import com.salesmanager.core.modules.utils.Encryption;


@Service("paymentService")
public class PaymentServiceImpl implements PaymentService {
	
	
	private static final Logger LOGGER = LoggerFactory.getLogger(PaymentServiceImpl.class);
	

	@Inject
	private MerchantConfigurationService merchantConfigurationService;
	
	@Inject
	private ModuleConfigurationService moduleConfigurationService;
	
	@Inject
	private TransactionService transactionService;
	
	@Inject
	private OrderService orderService;
	
	@Inject
	private CoreConfiguration coreConfiguration;
	
	@Inject
	@Resource(name="paymentModules")
	private Map<String,PaymentModule> paymentModules;
	
	@Inject
	private Encryption encryption;
	
	@Override
	public List<IntegrationModule> getPaymentMethods(MerchantStore store) throws ServiceException {
		
		List<IntegrationModule> modules =  moduleConfigurationService.getIntegrationModules(Constants.PAYMENT_MODULES);
		List<IntegrationModule> returnModules = new ArrayList<IntegrationModule>();
		
		for(IntegrationModule module : modules) {
			if(module.getRegionsSet().contains(store.getCountry().getIsoCode())
					|| module.getRegionsSet().contains("*")) {
				
				returnModules.add(module);
			}
		}
		
		return returnModules;
	}
	
	@Override
	public List<PaymentMethod> getAcceptedPaymentMethods(MerchantStore store) throws ServiceException {
		
		Map<String,IntegrationConfiguration> modules =  this.getPaymentModulesConfigured(store);

		List<PaymentMethod> returnModules = new ArrayList<PaymentMethod>();
		
		for(String module : modules.keySet()) {
			IntegrationConfiguration config = modules.get(module);
			if(config.isActive()) {
				
				IntegrationModule md = this.getPaymentMethodByCode(store, config.getModuleCode());
				if(md==null) {
					continue;
				}
				PaymentMethod paymentMethod = new PaymentMethod();
				
				paymentMethod.setDefaultSelected(config.isDefaultSelected());
				paymentMethod.setPaymentMethodCode(config.getModuleCode());
				paymentMethod.setModule(md);
				paymentMethod.setInformations(config);

				PaymentType type = PaymentType.fromString(md.getType());

				paymentMethod.setPaymentType(type);
				returnModules.add(paymentMethod);
			}
		}
		
		return returnModules;
		
		
	}
	
	@Override
	public IntegrationModule getPaymentMethodByType(MerchantStore store, String type) throws ServiceException {
		List<IntegrationModule> modules =  getPaymentMethods(store);

		for(IntegrationModule module : modules) {
			if(module.getModule().equals(type)) {
				
				return module;
			}
		}
		
		return null;
	}
	
	@Override
	public IntegrationModule getPaymentMethodByCode(MerchantStore store,
			String code) throws ServiceException {
		List<IntegrationModule> modules =  getPaymentMethods(store);

		for(IntegrationModule module : modules) {
			if(module.getCode().equals(code)) {
				
				return module;
			}
		}
		
		return null;
	}
	
	@Override
	public IntegrationConfiguration getPaymentConfiguration(String moduleCode, MerchantStore store) throws ServiceException {

		Validate.notNull(moduleCode,"Module code must not be null");
		Validate.notNull(store,"Store must not be null");
		
		String mod = moduleCode.toLowerCase();
		
		Map<String,IntegrationConfiguration> configuredModules = getPaymentModulesConfigured(store);
		if(configuredModules!=null) {
			for(String key : configuredModules.keySet()) {
				if(key.equals(mod)) {
					return configuredModules.get(key);	
				}
			}
		}
		
		return null;
		
	}
	

	
	@Override
	public Map<String,IntegrationConfiguration> getPaymentModulesConfigured(MerchantStore store) throws ServiceException {
		
		try {
		
			Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(Constants.PAYMENT_MODULES, store);
			if(merchantConfiguration!=null) {
				
				if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
					
					String decrypted = encryption.decrypt(merchantConfiguration.getValue());
					modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
					
					
				}
			}
			return modules;
		
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	}
	
	@Override
	public void savePaymentModuleConfiguration(IntegrationConfiguration configuration, MerchantStore store) throws ServiceException {
		
		//validate entries
		try {
			
			String moduleCode = configuration.getModuleCode();
			PaymentModule module = paymentModules.get(moduleCode);
			if(module==null) {
				throw new ServiceException("Payment module " + moduleCode + " does not exist");
			}
			module.validateModuleConfiguration(configuration, store);
			
		} catch (IntegrationException ie) {
			throw ie;
		}
		
		try {
			Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(Constants.PAYMENT_MODULES, store);
			if(merchantConfiguration!=null) {
				if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
					
					String decrypted = encryption.decrypt(merchantConfiguration.getValue());
					
					modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
				}
			} else {
				merchantConfiguration = new MerchantConfiguration();
				merchantConfiguration.setMerchantStore(store);
				merchantConfiguration.setKey(Constants.PAYMENT_MODULES);
			}
			modules.put(configuration.getModuleCode(), configuration);
			
			String configs =  ConfigurationModulesLoader.toJSONString(modules);
			
			String encrypted = encryption.encrypt(configs);
			merchantConfiguration.setValue(encrypted);
			
			merchantConfigurationService.saveOrUpdate(merchantConfiguration);
			
		} catch (Exception e) {
			throw new ServiceException(e);
		}
   }
	
	@Override
	public void removePaymentModuleConfiguration(String moduleCode, MerchantStore store) throws ServiceException {
		
		

		try {
			Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(Constants.PAYMENT_MODULES, store);
			if(merchantConfiguration!=null) {

				if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
					
					String decrypted = encryption.decrypt(merchantConfiguration.getValue());
					modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
				}
				
				modules.remove(moduleCode);
				String configs =  ConfigurationModulesLoader.toJSONString(modules);
				
				String encrypted = encryption.encrypt(configs);
				merchantConfiguration.setValue(encrypted);
				
				merchantConfigurationService.saveOrUpdate(merchantConfiguration);
				
				
			} 
			
			MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(moduleCode, store);
			
			if(configuration!=null) {//custom module

				merchantConfigurationService.delete(configuration);
			}

			
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	
	}
	

	


	@Override
	public Transaction processPayment(Customer customer,
			MerchantStore store, Payment payment, List<ShoppingCartItem> items, Order order)
			throws ServiceException {


		Validate.notNull(customer);
		Validate.notNull(store);
		Validate.notNull(payment);
		Validate.notNull(order);
		Validate.notNull(order.getTotal());
		
		payment.setCurrency(store.getCurrency());
		
		BigDecimal amount = order.getTotal();

		//must have a shipping module configured
		Map<String, IntegrationConfiguration> modules = this.getPaymentModulesConfigured(store);
		if(modules==null){
			throw new ServiceException("No payment module configured");
		}
		
		IntegrationConfiguration configuration = modules.get(payment.getModuleName());
		
		if(configuration==null) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not configured");
		}
		
		if(!configuration.isActive()) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not active");
		}
		
		String sTransactionType = configuration.getIntegrationKeys().get("transaction");
		if(sTransactionType==null) {
			sTransactionType = TransactionType.AUTHORIZECAPTURE.name();
		}
		
		try {
			TransactionType.valueOf(sTransactionType);
		} catch(IllegalArgumentException ie) {
			LOGGER.warn("Transaction type " + sTransactionType + " does noe exist, using default AUTHORIZECAPTURE");
			sTransactionType = "AUTHORIZECAPTURE";
		}

		

		if(sTransactionType.equals(TransactionType.AUTHORIZE.name())) {
			payment.setTransactionType(TransactionType.AUTHORIZE);
		} else {
			payment.setTransactionType(TransactionType.AUTHORIZECAPTURE);
		} 
		

		PaymentModule module = this.paymentModules.get(payment.getModuleName());
		
		if(module==null) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " does not exist");
		}
		
		if(payment instanceof CreditCardPayment && "true".equals(coreConfiguration.getProperty("VALIDATE_CREDIT_CARD"))) {
			CreditCardPayment creditCardPayment = (CreditCardPayment)payment;
			validateCreditCard(creditCardPayment.getCreditCardNumber(),creditCardPayment.getCreditCard(),creditCardPayment.getExpirationMonth(),creditCardPayment.getExpirationYear());
		}
		
		IntegrationModule integrationModule = getPaymentMethodByCode(store,payment.getModuleName());
		TransactionType transactionType = TransactionType.valueOf(sTransactionType);
		if(transactionType==null) {
			transactionType = payment.getTransactionType();
			if(transactionType.equals(TransactionType.CAPTURE.name())) {
				throw new ServiceException("This method does not allow to process capture transaction. Use processCapturePayment");
			}
		}
		
		Transaction transaction = null;
		if(transactionType == TransactionType.AUTHORIZE)  {
			transaction = module.authorize(store, customer, items, amount, payment, configuration, integrationModule);
		} else if(transactionType == TransactionType.AUTHORIZECAPTURE)  {
			transaction = module.authorizeAndCapture(store, customer, items, amount, payment, configuration, integrationModule);
		} else if(transactionType == TransactionType.INIT)  {
			transaction = module.initTransaction(store, customer, amount, payment, configuration, integrationModule);
		}


		if(transactionType != TransactionType.INIT) {
			transactionService.create(transaction);
		}
		
		if(transactionType == TransactionType.AUTHORIZECAPTURE)  {
			order.setStatus(OrderStatus.ORDERED);
			if(!payment.getPaymentType().name().equals(PaymentType.MONEYORDER.name())) {
				order.setStatus(OrderStatus.PROCESSED);
			}
		}

		return transaction;

		

	}
	
	@Override
	public PaymentModule getPaymentModule(String paymentModuleCode) throws ServiceException {
		return paymentModules.get(paymentModuleCode);
	}
	
	@Override
	public Transaction processCapturePayment(Order order, Customer customer,
			MerchantStore store)
			throws ServiceException {


		Validate.notNull(customer);
		Validate.notNull(store);
		Validate.notNull(order);

		

		//must have a shipping module configured
		Map<String, IntegrationConfiguration> modules = this.getPaymentModulesConfigured(store);
		if(modules==null){
			throw new ServiceException("No payment module configured");
		}
		
		IntegrationConfiguration configuration = modules.get(order.getPaymentModuleCode());
		
		if(configuration==null) {
			throw new ServiceException("Payment module " + order.getPaymentModuleCode() + " is not configured");
		}
		
		if(!configuration.isActive()) {
			throw new ServiceException("Payment module " + order.getPaymentModuleCode() + " is not active");
		}
		
		
		PaymentModule module = this.paymentModules.get(order.getPaymentModuleCode());
		
		if(module==null) {
			throw new ServiceException("Payment module " + order.getPaymentModuleCode() + " does not exist");
		}
		

		IntegrationModule integrationModule = getPaymentMethodByCode(store,order.getPaymentModuleCode());
		
		//TransactionType transactionType = payment.getTransactionType();

			//get the previous transaction
		Transaction trx = transactionService.getCapturableTransaction(order);
		if(trx==null) {
			throw new ServiceException("No capturable transaction for order id " + order.getId());
		}
		Transaction transaction = module.capture(store, customer, order, trx, configuration, integrationModule);
		transaction.setOrder(order);
		
		

		transactionService.create(transaction);
		
		
		OrderStatusHistory orderHistory = new OrderStatusHistory();
		orderHistory.setOrder(order);
		orderHistory.setStatus(OrderStatus.PROCESSED);
		orderHistory.setDateAdded(new Date());
		
		orderService.addOrderStatusHistory(order, orderHistory);
		
		order.setStatus(OrderStatus.PROCESSED);
		orderService.saveOrUpdate(order);

		return transaction;

		

	}

	@Override
	public Transaction processRefund(Order order, Customer customer,
			MerchantStore store, BigDecimal amount)
			throws ServiceException {
		
		
		Validate.notNull(customer);
		Validate.notNull(store);
		Validate.notNull(amount);
		Validate.notNull(order);
		Validate.notNull(order.getOrderTotal());
		
		
		BigDecimal orderTotal = order.getTotal();
		
		if(amount.doubleValue()>orderTotal.doubleValue()) {
			throw new ServiceException("Invalid amount, the refunded amount is greater than the total allowed");
		}

		
		String module = order.getPaymentModuleCode();
		Map<String, IntegrationConfiguration> modules = this.getPaymentModulesConfigured(store);
		if(modules==null){
			throw new ServiceException("No payment module configured");
		}
		
		IntegrationConfiguration configuration = modules.get(module);
		
		if(configuration==null) {
			throw new ServiceException("Payment module " + module + " is not configured");
		}
		
		PaymentModule paymentModule = this.paymentModules.get(module);
		
		if(paymentModule==null) {
			throw new ServiceException("Payment module " + paymentModule + " does not exist");
		}
		
		boolean partial = false;
		if(amount.doubleValue()!=order.getTotal().doubleValue()) {
			partial = true;
		}
		
		IntegrationModule integrationModule = getPaymentMethodByCode(store,module);
		
		//get the associated transaction
		Transaction refundable = transactionService.getRefundableTransaction(order);
		
		if(refundable==null) {
			throw new ServiceException("No refundable transaction for this order");
		}
		
		Transaction transaction = paymentModule.refund(partial, store, refundable, order, amount, configuration, integrationModule);
		transaction.setOrder(order);
		transactionService.create(transaction);
		
        OrderTotal refund = new OrderTotal();
        refund.setModule(Constants.OT_REFUND_MODULE_CODE);
        refund.setText(Constants.OT_REFUND_MODULE_CODE);
        refund.setTitle(Constants.OT_REFUND_MODULE_CODE);
        refund.setOrderTotalCode(Constants.OT_REFUND_MODULE_CODE);
        refund.setOrderTotalType(OrderTotalType.REFUND);
        refund.setValue(amount);
        refund.setSortOrder(100);
        refund.setOrder(order);
        
        order.getOrderTotal().add(refund);
        
		//update order total
		orderTotal = orderTotal.subtract(amount);
        
        //update ordertotal refund
        Set<OrderTotal> totals = order.getOrderTotal();
        for(OrderTotal total : totals) {
        	if(total.getModule().equals(Constants.OT_TOTAL_MODULE_CODE)) {
        		total.setValue(orderTotal);
        	}
        }

		

		order.setTotal(orderTotal);
		order.setStatus(OrderStatus.REFUNDED);
		
		
		
		OrderStatusHistory orderHistory = new OrderStatusHistory();
		orderHistory.setOrder(order);
		orderHistory.setStatus(OrderStatus.REFUNDED);
		orderHistory.setDateAdded(new Date());
        order.getOrderHistory().add(orderHistory);
        
        orderService.saveOrUpdate(order);

		return transaction;
	}
	
	@Override
	public void validateCreditCard(String number, CreditCardType creditCard, String month, String date)
	throws ServiceException {

		try {
			Integer.parseInt(month);
			Integer.parseInt(date);
		} catch (NumberFormatException nfe) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid date format","messages.error.creditcard.dateformat");
		}
		
		if (StringUtils.isBlank(number)) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
		}
		
		Matcher m = Pattern.compile("[^\\d\\s.-]").matcher(number);
		
		if (m.find()) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
		}
		
		Matcher matcher = Pattern.compile("[\\s.-]").matcher(number);
		
		number = matcher.replaceAll("");
		validateCreditCardDate(Integer.parseInt(month), Integer.parseInt(date));
		validateCreditCardNumber(number, creditCard);
	}

	private void validateCreditCardDate(int m, int y) throws ServiceException {
		java.util.Calendar cal = new java.util.GregorianCalendar();
		int monthNow = cal.get(java.util.Calendar.MONTH) + 1;
		int yearNow = cal.get(java.util.Calendar.YEAR);
		if (yearNow > y) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid date format","messages.error.creditcard.dateformat");
		}
		// OK, change implementation
		if (yearNow == y && monthNow > m) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid date format","messages.error.creditcard.dateformat");
		}
	
	}
	
	@Deprecated
	/**
	 * Use commons validator CreditCardValidator
	 * @param number
	 * @param creditCard
	 * @throws ServiceException
	 */
	private void validateCreditCardNumber(String number, CreditCardType creditCard)
	throws ServiceException {

		//TODO implement
		if(CreditCardType.MASTERCARD.equals(creditCard.name())) {
			if (number.length() != 16
					|| Integer.parseInt(number.substring(0, 2)) < 51
					|| Integer.parseInt(number.substring(0, 2)) > 55) {
				throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
			}
		}
		
		if(CreditCardType.VISA.equals(creditCard.name())) {
			if ((number.length() != 13 && number.length() != 16)
					|| Integer.parseInt(number.substring(0, 1)) != 4) {
				throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
			}
		}
		
		if(CreditCardType.AMEX.equals(creditCard.name())) {
			if (number.length() != 15
					|| (Integer.parseInt(number.substring(0, 2)) != 34 && Integer
							.parseInt(number.substring(0, 2)) != 37)) {
				throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
			}
		}
		
		if(CreditCardType.DINERS.equals(creditCard.name())) {
			if (number.length() != 14
					|| ((Integer.parseInt(number.substring(0, 2)) != 36 && Integer
							.parseInt(number.substring(0, 2)) != 38)
							&& Integer.parseInt(number.substring(0, 3)) < 300 || Integer
							.parseInt(number.substring(0, 3)) > 305)) {
				throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
			}
		}
		
		if(CreditCardType.DISCOVERY.equals(creditCard.name())) {
			if (number.length() != 16
					|| Integer.parseInt(number.substring(0, 5)) != 6011) {
				throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
			}
		}

		luhnValidate(number);
	}

	// The Luhn algorithm is basically a CRC type
	// system for checking the validity of an entry.
	// All major credit cards use numbers that will
	// pass the Luhn check. Also, all of them are based
	// on MOD 10.
	@Deprecated
	private void luhnValidate(String numberString)
			throws ServiceException {
		char[] charArray = numberString.toCharArray();
		int[] number = new int[charArray.length];
		int total = 0;
	
		for (int i = 0; i < charArray.length; i++) {
			number[i] = Character.getNumericValue(charArray[i]);
		}
	
		for (int i = number.length - 2; i > -1; i -= 2) {
			number[i] *= 2;
	
			if (number[i] > 9)
				number[i] -= 9;
		}

		for (int j : number) {
			total += j;
		}
	
		if (total % 10 != 0) {
			throw new ServiceException(ServiceException.EXCEPTION_VALIDATION,"Invalid card number","messages.error.creditcard.number");
		}
	
	}

	@Override
	public Transaction initTransaction(Order order, Customer customer, Payment payment, MerchantStore store) throws ServiceException {
		
		Validate.notNull(store);
		Validate.notNull(payment);
		Validate.notNull(order);
		Validate.notNull(order.getTotal());
		
		payment.setCurrency(store.getCurrency());
		
		BigDecimal amount = order.getTotal();

		//must have a shipping module configured
		Map<String, IntegrationConfiguration> modules = this.getPaymentModulesConfigured(store);
		if(modules==null){
			throw new ServiceException("No payment module configured");
		}
		
		IntegrationConfiguration configuration = modules.get(payment.getModuleName());
		
		if(configuration==null) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not configured");
		}
		
		if(!configuration.isActive()) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not active");
		}
		
		PaymentModule module = this.paymentModules.get(order.getPaymentModuleCode());
		
		if(module==null) {
			throw new ServiceException("Payment module " + order.getPaymentModuleCode() + " does not exist");
		}
		
		IntegrationModule integrationModule = getPaymentMethodByCode(store,payment.getModuleName());

		return module.initTransaction(store, customer, amount, payment, configuration, integrationModule);
	}

	@Override
	public Transaction initTransaction(Customer customer, Payment payment, MerchantStore store) throws ServiceException {

		Validate.notNull(store);
		Validate.notNull(payment);
		Validate.notNull(payment.getAmount());
		
		payment.setCurrency(store.getCurrency());
		
		BigDecimal amount = payment.getAmount();

		//must have a shipping module configured
		Map<String, IntegrationConfiguration> modules = this.getPaymentModulesConfigured(store);
		if(modules==null){
			throw new ServiceException("No payment module configured");
		}
		
		IntegrationConfiguration configuration = modules.get(payment.getModuleName());
		
		if(configuration==null) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not configured");
		}
		
		if(!configuration.isActive()) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " is not active");
		}
		
		PaymentModule module = this.paymentModules.get(payment.getModuleName());
		
		if(module==null) {
			throw new ServiceException("Payment module " + payment.getModuleName() + " does not exist");
		}
		
		IntegrationModule integrationModule = getPaymentMethodByCode(store,payment.getModuleName());
		
		Transaction transaction = module.initTransaction(store, customer, amount, payment, configuration, integrationModule);
		
		transactionService.save(transaction);

		return transaction;
	}


	


}



```
