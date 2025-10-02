# StripePayment.java

## Review

## 1. Summary  

**Purpose** – `StripePayment` is a concrete implementation of the `PaymentModule` interface that delegates all payment‑related operations to Stripe’s REST API. It handles the following high‑level actions:

| Action | What it does |
|--------|--------------|
| `validateModuleConfiguration` | Ensures that the integration keys (`secretKey`, `publishableKey`) are present. |
| `initTransaction` | Builds a lightweight `Transaction` instance containing the amount, payment type and some meta data (the publishable key). |
| `authorize` | Creates a *pre‑auth* Stripe charge (`capture=false`) and records the resulting charge ID and status in the transaction. |
| `capture` | Takes a previously authorized charge (identified by the charge ID stored in the transaction) and captures it. |
| `authorizeAndCapture` | Creates a single charge with `capture=true` (i.e., a full immediate capture). |
| `refund` | Issues a partial or full refund on a charge and stores the resulting refund id. |
| `buildException` | Maps various Stripe exceptions into domain‑specific `IntegrationException`s. |

The class is wired through CDI (`@Inject`) for `ProductPriceUtils`, and uses the Stripe Java SDK (`com.stripe.*`). It also depends on Apache Commons Lang for string utilities and on SLF4J for logging.

---

## 2. Detailed Description  

### 2.1 Core Flow

1. **Configuration Validation** – Before any operation, the integration keys are verified to be non‑null/blank.  
2. **Transaction Creation** – All public methods (`authorize`, `capture`, …) create a `Transaction` object. The amount, dates, payment type, and transaction type are populated.  
3. **Stripe API Interaction**  
   * `authorize` → `Charge.create` with `capture=false`.  
   * `capture` → `Charge.retrieve` → `capture()`.  
   * `authorizeAndCapture` → `Charge.create` with `capture=true`.  
   * `refund` → `Refund.create` using the original charge ID.  
4. **Exception Handling** – All Stripe calls are wrapped in a `try/catch`. If an exception occurs, it is passed to `buildException`, which maps Stripe exception types to `IntegrationException` instances with user‑friendly error codes/messages.  

### 2.2 Design Choices & Assumptions

| Choice | Reason / Assumption |
|--------|---------------------|
| `BigDecimal` → String → replace “.” → Integer amount | Stripe expects amount in the smallest currency unit (cents). The implementation converts via string manipulation; a more robust approach would use `BigDecimal.multiply(new BigDecimal(100))`. |
| Storing *token* in `TRANSACTIONID` | The token is a one‑time payment source. The real transaction ID (charge id) is stored in `TRNORDERNUMBER`. |
| `TransactionDetails` map keys | Hard‑coded string keys (`TRANSACTIONID`, `TRNAPPROVED`, etc.) with no constants. This can lead to typos. |
| `buildException` covers many Stripe exception types | Attempts to translate Stripe error codes into domain‑specific error types; however, the mapping is partially duplicated and not exhaustive. |
| No idempotency handling | The code does not use Stripe’s idempotency key feature, potentially causing duplicate charges on retry. |
| No unit testing evidence | Implementation is tightly coupled to Stripe’s SDK, making it hard to test in isolation. |

### 2.3 Architecture

The module follows a **service‑layer** pattern where the interface (`PaymentModule`) decouples the business logic from the concrete provider (Stripe). This allows future providers (e.g., PayPal, Authorize.Net) to be added without changing the rest of the system.

The code is **state‑free** – each method constructs a new `Transaction` object; no shared mutable state is present. Dependencies are injected (`ProductPriceUtils`) which makes it easier to mock in tests.

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `validateModuleConfiguration` | Ensures required keys exist. | `IntegrationConfiguration`, `MerchantStore` | `void` | Throws `IntegrationException` if validation fails |
| `initTransaction` | Creates an initial `Transaction` (no Stripe call). | `MerchantStore`, `Customer`, `BigDecimal amount`, `Payment`, `IntegrationConfiguration`, `IntegrationModule` | `Transaction` | None |
| `authorize` | Pre‑authorises a charge (no capture). | `MerchantStore`, `Customer`, `List<ShoppingCartItem>`, `BigDecimal amount`, `Payment`, `IntegrationConfiguration`, `IntegrationModule` | `Transaction` | Creates a Stripe charge, populates `Transaction` |
| `capture` | Captures a previously authorised charge. | `MerchantStore`, `Customer`, `Order`, `Transaction capturableTransaction`, `IntegrationConfiguration`, `IntegrationModule` | `Transaction` | Calls `Charge.capture()` |
| `authorizeAndCapture` | One‑step charge & capture. | Same as `authorize` | `Transaction` | Creates a charge with `capture=true` |
| `refund` | Issues a (partial or full) refund. | `boolean partial`, `MerchantStore`, `Transaction`, `Order`, `BigDecimal amount`, `IntegrationConfiguration`, `IntegrationModule` | `Transaction` | Calls `Refund.create()` |
| `buildException` | Maps any exception into a domain `IntegrationException`. | `Exception ex` | `IntegrationException` | None (purely transform) |

**Utility Methods**

* None – the class relies on injected services and Stripe SDK directly.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `com.stripe:stripe-java` | Third‑party | Stripe SDK for REST interactions |
| `org.apache.commons:commons-lang3` | Third‑party | For `StringUtils` and `Validate` |
| `org.slf4j:slf4j-api` | Third‑party | Logging |
| `com.salesmanager.core.*` | In‑house | Domain models (`Customer`, `Order`, `Payment`, `Transaction`, etc.) |
| `javax.inject.Inject` | Standard | CDI / JSR‑330 injection |
| `java.util.*` | Standard | Collections, dates |

No platform‑specific dependencies are visible, but the class is tied to the Stripe Java library, which itself depends on Java 8+.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality & Maintainability  

| Issue | Impact | Fix / Suggestion |
|-------|--------|------------------|
| **String manipulation for amount** | Potential bugs with locales, negative amounts, precision loss | Use `BigDecimal` multiplication: `int cents = amount.multiply(new BigDecimal(100)).intValueExact();` |
| **Hard‑coded map keys** | Typos lead to silent failures | Define constants or an enum for `TRANSACTIONID`, `TRNAPPROVED`, … |
| **Token stored as `TRANSACTIONID`** | Confusing – transaction ID should be the Stripe charge ID | Store charge ID in `TRANSACTIONID` or create a dedicated key |
| **Error mapping duplication** | Hard to maintain, hard‑coded error codes | Centralise error mapping, use a lookup table or strategy pattern |
| **No idempotency** | Duplicate charges on network retry | Pass a unique idempotency key in `Charge.create()` and `Refund.create()` |
| **Partial refund not used** | `partial` flag ignored | Use `partial` to determine the refund amount or throw if unsupported |
| **Logging** | Only error messages are logged, stack trace lost in some branches | Log full stack trace (`LOGGER.error("...", ex);`) |
| **Missing null checks** | Possible `NullPointerException` in some branches (e.g., `transaction.getTransactionDetails()`) | Add defensive checks or use `Objects.requireNonNull` |
| **Unit testing** | Tight coupling to Stripe SDK makes unit testing hard | Extract Stripe calls into a wrapper interface and mock it in tests |

### 5.2 Functional Edge Cases  

* **Currency with subunits other than 100** (e.g., Japanese yen has no subunits). The current `replace(".")` logic will fail for currencies that don’t have a cent component.  
* **Large amounts** could overflow an `int` if converted via `String.replace`.  
* **Network failures** or API downtime are swallowed into generic `IntegrationException` without retry logic.  
* **Refund**: The code always sets the transaction amount to `order.getTotal()` regardless of the actual refunded amount; this can mislead reporting.  
* **Transaction ID vs Charge ID**: `TransactionID` holds the token, while `TRNORDERNUMBER` holds the charge id. When later operations (e.g., capture) look for a charge id, they may get a token, causing `IllegalArgumentException` or `InvalidRequestException`.

### 5.3 Future Enhancements  

1. **Refactor amount handling** – use a dedicated `Money` utility that correctly converts `BigDecimal` to the smallest unit across currencies.  
2. **Idempotency support** – expose an `idempotencyKey` parameter in public methods.  
3. **Unified error handling** – move all exception mapping into a separate `StripeErrorMapper` class.  
4. **Better transaction metadata** – create a `StripeTransactionDetails` DTO instead of a raw map.  
5. **Unit tests** – mock the Stripe API via a thin wrapper; test each branch of `buildException`.  
6. **Partial refunds** – honor the `partial` flag and pass the correct amount to `Refund.create()`.  
7. **Logging enhancements** – include correlation IDs and full stack traces for easier debugging.  

---

### 5.4 Summary of Strengths  

* Clear separation between configuration validation and transaction logic.  
* Uses dependency injection for price formatting.  
* Provides comprehensive exception mapping for end‑user error messaging.  

### 5.5 Summary of Weaknesses  

* Inconsistent handling of transaction identifiers.  
* Fragile amount conversion logic.  
* Hard‑coded string keys and duplicated error handling.  
* No support for idempotency, partial refunds, or robust error handling.  

By addressing the above issues, the module would become more reliable, maintainable, and easier to test, while also offering better support for edge cases and future payment providers.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.payment.impl;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.utils.ProductPriceUtils;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.payments.Payment;
import com.salesmanager.core.model.payments.PaymentType;
import com.salesmanager.core.model.payments.Transaction;
import com.salesmanager.core.model.payments.TransactionType;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.payment.model.PaymentModule;
import com.stripe.Stripe;
// import com.stripe.exception.APIConnectionException;
import com.stripe.exception.AuthenticationException;
import com.stripe.exception.CardException;
import com.stripe.exception.InvalidRequestException;
import com.stripe.exception.StripeException;
import com.stripe.model.Charge;
import com.stripe.model.Refund;

public class StripePayment implements PaymentModule {
	
	@Inject
	private ProductPriceUtils productPriceUtils;

	
	private final static String AUTHORIZATION = "Authorization";
	private final static String TRANSACTION = "Transaction";
	
	private static final Logger LOGGER = LoggerFactory.getLogger(StripePayment.class);
	
	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		
		
		List<String> errorFields = null;
		
		
		Map<String,String> keys = integrationConfiguration.getIntegrationKeys();
		
		//validate integrationKeys['secretKey']
		if(keys==null || StringUtils.isBlank(keys.get("secretKey"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("secretKey");
		}
		
		//validate integrationKeys['publishableKey']
		if(keys==null || StringUtils.isBlank(keys.get("publishableKey"))) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("publishableKey");
		}
		
		
		if(errorFields!=null) {
			IntegrationException ex = new IntegrationException(IntegrationException.ERROR_VALIDATION_SAVE);
			ex.setErrorFields(errorFields);
			throw ex;
			
		}
		
		
		
	}


	@Override
	public Transaction initTransaction(MerchantStore store, Customer customer,
			BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {
      Validate.notNull(configuration,"Configuration cannot be null");
      String publicKey = configuration.getIntegrationKeys().get("publishableKey");
      Validate.notNull(publicKey,"Publishable key not found in configuration");

      Transaction transaction = new Transaction();
      transaction.setAmount(amount);
      transaction.setDetails(publicKey);
      transaction.setPaymentType(payment.getPaymentType());
      transaction.setTransactionDate(new Date());
      transaction.setTransactionType(payment.getTransactionType());
      
      return transaction;
	}

	@Override
	public Transaction authorize(MerchantStore store, Customer customer,
			List<ShoppingCartItem> items, BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {

		Transaction transaction = new Transaction();
		try {
			

			String apiKey = configuration.getIntegrationKeys().get("secretKey");

			if(payment.getPaymentMetaData()==null || StringUtils.isBlank(apiKey)) {
				IntegrationException te = new IntegrationException(
						"Can't process Stripe, missing payment.metaData");
				te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
				te.setMessageCode("message.payment.error");
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				throw te;
			}
			
			/**
			 * this is send by stripe from tokenization ui
			 */
			String token = payment.getPaymentMetaData().get("stripe_token");
			
			if(StringUtils.isBlank(token)) {
				IntegrationException te = new IntegrationException(
						"Can't process Stripe, missing stripe token");
				te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
				te.setMessageCode("message.payment.error");
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				throw te;
			}
			

			String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
			
			//stripe does not support floating point
			//so amnt * 100 or remove floating point
			//553.47 = 55347
			
			String strAmount = String.valueOf(amnt);
			strAmount = strAmount.replace(".","");
			
			Map<String, Object> chargeParams = new HashMap<String, Object>();
			chargeParams.put("amount", strAmount);
			chargeParams.put("capture", false);
			chargeParams.put("currency", store.getCurrency().getCode());
			chargeParams.put("source", token); // obtained with Stripe.js
			chargeParams.put("description", new StringBuilder().append(TRANSACTION).append(" - ").append(store.getStorename()).toString());
			
			Stripe.apiKey = apiKey;
			
			
			Charge ch = Charge.create(chargeParams);

			//Map<String,String> metadata = ch.getMetadata();
			
			
			transaction.setAmount(amount);
			//transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.AUTHORIZE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", token);
			transaction.getTransactionDetails().put("TRNAPPROVED", ch.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", ch.getId());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);
			
		} catch (Exception e) {
			
			throw buildException(e);

		} 
		
		return transaction;

		
	}

	@Override
	public Transaction capture(MerchantStore store, Customer customer,
			Order order, Transaction capturableTransaction,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {


			Transaction transaction = new Transaction();
			try {
				
				
				String apiKey = configuration.getIntegrationKeys().get("secretKey");

				if(StringUtils.isBlank(apiKey)) {
					IntegrationException te = new IntegrationException(
							"Can't process Stripe, missing payment.metaData");
					te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
					te.setMessageCode("message.payment.error");
					te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
					throw te;
				}
				
				String chargeId = capturableTransaction.getTransactionDetails().get("TRNORDERNUMBER");
				
				if(StringUtils.isBlank(chargeId)) {
					IntegrationException te = new IntegrationException(
							"Can't process Stripe capture, missing TRNORDERNUMBER");
					te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
					te.setMessageCode("message.payment.error");
					te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
					throw te;
				}
				

				Stripe.apiKey = apiKey;
				
				Charge ch = Charge.retrieve(chargeId);
				ch.capture();
				
				
				transaction.setAmount(order.getTotal());
				transaction.setOrder(order);
				transaction.setTransactionDate(new Date());
				transaction.setTransactionType(TransactionType.CAPTURE);
				transaction.setPaymentType(PaymentType.CREDITCARD);
				transaction.getTransactionDetails().put("TRANSACTIONID", capturableTransaction.getTransactionDetails().get("TRANSACTIONID"));
				transaction.getTransactionDetails().put("TRNAPPROVED", ch.getStatus());
				transaction.getTransactionDetails().put("TRNORDERNUMBER", ch.getId());
				transaction.getTransactionDetails().put("MESSAGETEXT", null);
				
				//authorize a preauth 


				return transaction;
				
			} catch (Exception e) {
				
				throw buildException(e);

			}  

	}

	@Override
	public Transaction authorizeAndCapture(MerchantStore store, Customer customer,
			List<ShoppingCartItem> items, BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {

		String apiKey = configuration.getIntegrationKeys().get("secretKey");

		if(payment.getPaymentMetaData()==null || StringUtils.isBlank(apiKey)) {
			IntegrationException te = new IntegrationException(
					"Can't process Stripe, missing payment.metaData");
			te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
			te.setMessageCode("message.payment.error");
			te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
			throw te;
		}
		
		String token = payment.getPaymentMetaData().get("stripe_token");
		if(StringUtils.isBlank(token)) { //possibly from api
		  token = payment.getPaymentMetaData().get("paymentToken");//from api
		}
		
		if(StringUtils.isBlank(token)) {
			IntegrationException te = new IntegrationException(
					"Can't process Stripe, missing stripe token");
			te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
			te.setMessageCode("message.payment.error");
			te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
			throw te;
		}
		


		Transaction transaction = new Transaction();
		try {
			
			String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
			
			//stripe does not support floating point
			//so amnt * 100 or remove floating point
			//553.47 = 55347
			
		
			String strAmount = String.valueOf(amnt);
			strAmount = strAmount.replace(".","");
			
			Map<String, Object> chargeParams = new HashMap<String, Object>();
			chargeParams.put("amount", strAmount);
			chargeParams.put("capture", true);
			chargeParams.put("currency", store.getCurrency().getCode());
			chargeParams.put("source", token); // obtained with Stripe.js
			chargeParams.put("description", new StringBuilder().append(TRANSACTION).append(" - ").append(store.getStorename()).toString());
			
			Stripe.apiKey = apiKey;
			
			
			Charge ch = Charge.create(chargeParams);
	
			//Map<String,String> metadata = ch.getMetadata();
			
			
			transaction.setAmount(amount);
			//transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.AUTHORIZECAPTURE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", token);
			transaction.getTransactionDetails().put("TRNAPPROVED", ch.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", ch.getId());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);
			
		} catch (Exception e) {
			
			throw buildException(e);
	
		} 
		
		return transaction;  
		
	}

	@Override
	public Transaction refund(boolean partial, MerchantStore store, Transaction transaction,
			Order order, BigDecimal amount,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {
		

		
		String apiKey = configuration.getIntegrationKeys().get("secretKey");

		if(StringUtils.isBlank(apiKey)) {
			IntegrationException te = new IntegrationException(
					"Can't process Stripe, missing payment.metaData");
			te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
			te.setMessageCode("message.payment.error");
			te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
			throw te;
		}

		try {
		

			String trnID = transaction.getTransactionDetails().get("TRNORDERNUMBER");
			
			String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
			
			Stripe.apiKey = apiKey;
			
			//stripe does not support floating point
			//so amnt * 100 or remove floating point
			//553.47 = 55347
			
			String strAmount = String.valueOf(amnt);
			strAmount = strAmount.replace(".","");

			Charge ch = Charge.retrieve(trnID);

			Map<String, Object> params = new HashMap<>();
			params.put("charge", ch.getId());
			params.put("amount", strAmount);
			Refund re = Refund.create(params);

			transaction = new Transaction();
			transaction.setAmount(order.getTotal());
			transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.CAPTURE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", transaction.getTransactionDetails().get("TRANSACTIONID"));
			transaction.getTransactionDetails().put("TRNAPPROVED", re.getReason());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", re.getId());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);

			return transaction;

			
		} catch(Exception e) {
			
			throw buildException(e);

		} 
		
		
		
	}
	
	private IntegrationException buildException(Exception ex) {
		
		
	if(ex instanceof CardException) {
		  CardException e = (CardException)ex;
		  // Since it's a decline, CardException will be caught
		  //System.out.println("Status is: " + e.getCode());
		  //System.out.println("Message is: " + e.getMessage());
		  
		  
			/**
			 * 
				invalid_number 	The card number is not a valid credit card number.
				invalid_expiry_month 	The card's expiration month is invalid.
				invalid_expiry_year 	The card's expiration year is invalid.
				invalid_cvc 	The card's security code is invalid.
				incorrect_number 	The card number is incorrect.
				expired_card 	The card has expired.
				incorrect_cvc 	The card's security code is incorrect.
				incorrect_zip 	The card's zip code failed validation.
				card_declined 	The card was declined.
				missing 	There is no card on a customer that is being charged.
				processing_error 	An error occurred while processing the card.
				rate_limit 	An error occurred due to requests hitting the API too quickly. Please let us know if you're consistently running into this error.
			 */
		
			
			String declineCode = e.getDeclineCode();
			
			if("card_declined".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_PAYMENT_DECLINED);
				te.setMessageCode("message.payment.declined");
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				return te;
			}
			
			if("invalid_number".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.number");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			if("invalid_expiry_month".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.dateformat");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			if("invalid_expiry_year".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.dateformat");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			if("invalid_cvc".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.cvc");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			if("incorrect_number".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.number");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			if("incorrect_cvc".equals(declineCode)) {
				IntegrationException te = new IntegrationException(
						"Can't process stripe message " + e.getMessage());
				te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
				te.setMessageCode("messages.error.creditcard.cvc");
				te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
				return te;
			}
			
			//nothing good so create generic error
			IntegrationException te = new IntegrationException(
					"Can't process stripe card  " + e.getMessage());
			te.setExceptionType(IntegrationException.EXCEPTION_VALIDATION);
			te.setMessageCode("messages.error.creditcard.number");
			te.setErrorCode(IntegrationException.EXCEPTION_VALIDATION);
			return te;
		

		  
	} else if (ex instanceof InvalidRequestException) {
		LOGGER.error("InvalidRequest error with stripe", ex.getMessage());
		InvalidRequestException e =(InvalidRequestException)ex;
		IntegrationException te = new IntegrationException(
				"Can't process Stripe, missing invalid payment parameters");
		te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
		te.setMessageCode("messages.error.creditcard.number");
		te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
		return te;
		
	} else if (ex instanceof AuthenticationException) {
		LOGGER.error("Authentication error with stripe", ex.getMessage());
		AuthenticationException e = (AuthenticationException)ex;
		  // Authentication with Stripe's API failed
		  // (maybe you changed API keys recently)
		IntegrationException te = new IntegrationException(
				"Can't process Stripe, missing invalid payment parameters");
		te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
		te.setMessageCode("message.payment.error");
		te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
		return te;
		
	} /*else if (ex instanceof APIConnectionException) { // DEPRECATED THIS EXCEPTION TYPE
		LOGGER.error("API connection error with stripe", ex.getMessage());
		APIConnectionException e = (APIConnectionException)ex;
		  // Network communication with Stripe failed
		IntegrationException te = new IntegrationException(
				"Can't process Stripe, missing invalid payment parameters");
		te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
		te.setMessageCode("message.payment.error");
		te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
		return te;
	} */else if (ex instanceof StripeException) {
		LOGGER.error("Error with stripe", ex.getMessage());
		StripeException e = (StripeException)ex;
		  // Display a very generic error to the user, and maybe send
		  // yourself an email
		IntegrationException te = new IntegrationException(
				"Can't process Stripe authorize, missing invalid payment parameters");
		te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
		te.setMessageCode("message.payment.error");
		te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
		return te;
		
		

	} else if (ex instanceof Exception) {
		LOGGER.error("Stripe module error", ex.getMessage());
		if(ex instanceof IntegrationException) {
			return (IntegrationException)ex;
		} else {
			IntegrationException te = new IntegrationException(
					"Can't process Stripe authorize, exception", ex);
			te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
			te.setMessageCode("message.payment.error");
			te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
			return te;
		}


	} else {
		LOGGER.error("Stripe module error", ex.getMessage());
		IntegrationException te = new IntegrationException(
				"Can't process Stripe authorize, exception", ex);
		te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
		te.setMessageCode("message.payment.error");
		te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
		return te;
	}

	}
	
	



}



```
