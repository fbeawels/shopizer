# Stripe3Payment.java

## Review

## 1. Summary

**Purpose**  
`Stripe3Payment` implements the `PaymentModule` interface for a 3‑D Secure Stripe integration in a sales‑manager application.  
It is responsible for:

| Stage | Method | Action |
|-------|--------|--------|
| Initialisation | `initTransaction` | Creates a `PaymentIntent` (manual capture) and records its details in a `Transaction`. |
| Configuration | `validateModuleConfiguration` | Verifies that the required API keys are present. |
| Authorisation | `authorize` | Uses a client‑generated `stripe_token` to look up a `PaymentIntent` and records the transaction. |
| Capture | `capture` | Executes the manual capture on an existing `PaymentIntent`. |
| Authorise & Capture | `authorizeAndCapture` | Performs both authorisation and capture in a single call. |
| Refund | `refund` | Issues a refund against a `PaymentIntent`. |
| Exception handling | `buildException` | Translates Stripe SDK exceptions into the application’s `IntegrationException`. |

The class relies on the Stripe Java SDK, a utility for formatting amounts (`ProductPriceUtils`), and a set of domain models (`Transaction`, `Payment`, etc.).

**Design patterns & libraries**  
* Dependency Injection – `@Inject` for `ProductPriceUtils`.  
* Factory‑like behaviour – the class acts as a factory for `Transaction` objects.  
* Exception translation – custom exception wrapping.  
* Uses **Stripe’s PaymentIntent** API (manual capture) to support 3‑D Secure flows.

---

## 2. Detailed Description

### Core Flow

1. **Configuration**  
   `validateModuleConfiguration` checks that the integration keys (`secretKey`, `publishableKey`) are supplied.  
   The key map is read by every transaction method to set `Stripe.apiKey`.

2. **Transaction Creation**  
   - `initTransaction` constructs a `PaymentIntent` with manual capture.  
   - `authorize` retrieves an existing `PaymentIntent` (identified by `stripe_token` coming from the front‑end).  
   - `authorizeAndCapture` retrieves the intent and immediately captures it.  

3. **Capture**  
   `capture` retrieves the intent using the ID stored in the original transaction, builds a `PaymentIntentCaptureParams` (amount, statement descriptor) and calls `paymentIntent.capture(params)`.

4. **Refund**  
   `refund` pulls the intent ID from the transaction, constructs a `Refund` using the intent ID and amount, and then creates a new `Transaction` to record the refund.

5. **Exception Handling**  
   Every public method catches a generic `Exception`, delegates to `buildException`, and re‑throws the returned `IntegrationException`.  
   `buildException` contains exhaustive handling of Stripe‑specific exceptions (`CardException`, `InvalidRequestException`, etc.) and maps them to `IntegrationException` with specific error codes and message keys.

### Assumptions & Constraints

| Item | Assumption / Constraint |
|------|------------------------|
| Amount formatting | Stripe expects an integer amount in the smallest currency unit (cents). The code removes the decimal point from the string representation of a `BigDecimal`. |
| API Key | Only one secret key is required; the publishable key is validated but never used in this module. |
| Thread‑safety | `Stripe.apiKey` is a static field; setting it in each method is not thread‑safe if the application is multi‑threaded. |
| Error handling | The module assumes that any Stripe exception is recoverable via the custom `IntegrationException`. |
| `Transaction` details | Keys (`TRANSACTIONID`, `TRNAPPROVED`, etc.) are hard‑coded; no constants are used. |
| `PaymentIntent` ID | Stored as `TRNORDERNUMBER`. The naming is confusing because it is an intent ID, not a charge ID. |

### Architecture & Design Choices

* The class is tightly coupled to Stripe’s SDK and the domain models.  
* All stateful information is passed through `IntegrationConfiguration` and `IntegrationModule`.  
* Repeated code (API key retrieval, amount conversion, exception handling) could be extracted into reusable utilities.  
* The use of `Stripe.apiKey` as a global static is a design smell; a per‑instance client would be safer.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `initTransaction` | Creates a `PaymentIntent` (manual capture) and populates a `Transaction`. | `MerchantStore store, Customer customer, BigDecimal amount, Payment payment, IntegrationConfiguration configuration, IntegrationModule module` | `Transaction` | Creates a Stripe `PaymentIntent` (network I/O). |
| `validateModuleConfiguration` | Ensures secret and publishable keys are present. | `IntegrationConfiguration integrationConfiguration, MerchantStore store` | throws `IntegrationException` if invalid | None |
| `authorize` | Looks up an existing `PaymentIntent` (via `stripe_token`) and records the transaction. | `MerchantStore store, Customer customer, List<ShoppingCartItem> items, BigDecimal amount, Payment payment, IntegrationConfiguration configuration, IntegrationModule module` | `Transaction` | Calls `PaymentIntent.retrieve`. |
| `capture` | Executes manual capture of a previously authorized `PaymentIntent`. | `MerchantStore store, Customer customer, Order order, Transaction capturableTransaction, IntegrationConfiguration configuration, IntegrationModule module` | `Transaction` | Calls `PaymentIntent.capture`. |
| `authorizeAndCapture` | Retrieves the intent and captures it in one call. | `MerchantStore store, Customer customer, List<ShoppingCartItem> items, BigDecimal amount, Payment payment, IntegrationConfiguration configuration, IntegrationModule module` | `Transaction` | Calls `PaymentIntent.retrieve` and `capture`. |
| `refund` | Creates a refund for a completed transaction. | `boolean partial, MerchantStore store, Transaction transaction, Order order, BigDecimal amount, IntegrationConfiguration configuration, IntegrationModule module` | `Transaction` | Calls `Refund.create`. |
| `buildException` | Maps Stripe exceptions to `IntegrationException` with appropriate codes/message keys. | `Exception ex` | `IntegrationException` | None |

### Reusable/Utility Methods

* `buildException` is the sole reusable helper inside this class; however, it contains a lot of duplicated logic (exception type checks, message construction) that could be extracted into a dedicated `StripeExceptionMapper`.

---

## 4. Dependencies

| Library / API | Category | Notes |
|---------------|----------|-------|
| `com.stripe.*` (Stripe Java SDK) | Third‑party | Handles PaymentIntent, Refund, and exception classes. |
| `com.salesmanager.core.*` | Project internal | Domain models (`Transaction`, `Payment`, etc.), utilities, and integration interfaces. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | Simple string utilities. |
| `org.slf4j.Logger` | Logging | Standard logging. |
| `javax.inject.Inject` | DI framework (likely CDI or Guice) | Injects `ProductPriceUtils`. |

All dependencies are standard for a Java EE / Spring application.

---

## 5. Additional Notes & Recommendations

### 5.1 Code Quality & Maintainability

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded string keys** (`"TRANSACTIONID"`, `"TRNAPPROVED"`, etc.) | Error‑prone, hard to refactor | Define constants in a dedicated class or enum. |
| **Duplicate code** (API key retrieval, amount conversion, exception mapping) | Code bloat, hard to maintain | Extract into helper methods or a separate `StripeUtils` class. |
| **`Stripe.apiKey` is static** | Thread‑safety risk in a multi‑threaded environment | Use a per‑instance client (e.g., `new ApiKey("sk_...")` if SDK supports) or ensure external synchronization. |
| **`ProductPriceUtils` usage** | Indirectly formats amounts; unclear if it strips currency symbols | Replace with a simple `BigDecimal` utility that multiplies by 100 and rounds to `long`. |
| **`buildException`** | Very verbose, many branches; potential for missed exception types | Refactor into a map of `Exception` class → handler lambda, or use a third‑party error‑mapping library. |
| **Logging** | Only error logs are emitted; debug logs for request/response could help troubleshooting | Add conditional debug logging with request/response bodies (mask sensitive data). |
| **Transaction type for refunds** | Uses `TransactionType.CAPTURE` instead of `REFUND` | Use the correct enum value to avoid confusion downstream. |
| **Refund implementation** | Uses `re.getReason()` as approval flag, but `Reason` is not a status; also sets `TRANSACTIONID` incorrectly | Capture `re.getId()` and map `re.getStatus()` or `re.getAmount()` appropriately. |
| **Error handling in `refund`** | The new `Transaction` uses the newly created object's map, which is empty | Copy the relevant details from the original transaction. |
| **Method names** | `initTransaction` and `authorizeAndCapture` both create/modify intents; naming could be clearer | e.g., `createPaymentIntent`, `authorizePaymentIntent`, `capturePaymentIntent`. |
| **JavaDoc** | No documentation | Add Javadoc for public methods and the class to aid future developers. |
| **Unit Tests** | Not provided | Write tests for each public method, mocking Stripe responses. |

### 5.2 Edge Cases & Failure Modes

| Scenario | Current Behavior | Recommendation |
|----------|------------------|----------------|
| Amount has more than two decimal places | The string conversion may truncate or produce wrong value | Use `BigDecimal.setScale(2, RoundingMode.HALF_UP)` and multiply by 100. |
| Amount contains thousand separators (e.g., “1 000.00”) | `replace(".", "")` leaves the comma or space, causing `NumberFormatException` | Clean the string first or perform arithmetic on `BigDecimal`. |
| Network failure or API rate limiting | `APIConnectionException` is commented out; may be swallowed as generic `StripeException` | Handle network exceptions separately; implement exponential backoff. |
| Missing `stripe_token` in `payment.getPaymentMetaData()` | Throws `IntegrationException` with message “missing stripe token” | Provide a clearer error key or propagate the underlying exception. |
| Currency that doesn’t use cents (e.g., JPY) | Still multiplies by 100 incorrectly | Detect currency and avoid scaling if not needed. |

### 5.3 Potential Enhancements

1. **Centralize amount handling** – create a `CurrencyUtils` that converts `BigDecimal` amounts to the integer representation expected by Stripe, respecting currency precision.  
2. **Use a Stripe client wrapper** – encapsulate all API calls inside a wrapper that sets the API key once per request context, avoiding static mutation.  
3. **Leverage Stripe’s Webhook support** – instead of manually populating transaction details, rely on webhook events for idempotent updates.  
4. **Add configuration caching** – load integration keys from a secure vault and cache them for the duration of the request.  
5. **Improve error reporting** – expose Stripe’s `id` and `type` in the `IntegrationException` payload for easier debugging by support staff.  
6. **Internationalisation** – use the returned message keys (`messages.error.creditcard.*`) in the UI layer for consistent localisation.  

---

### 5.4 Quick Refactor Sample

```java
public static final class StripeKeys {
    public static final String TRANSACTION_ID = "TRANSACTIONID";
    public static final String TRN_APPROVED = "TRNAPPROVED";
    public static final String TRN_ORDER_NUMBER = "TRNORDERNUMBER";
    // … add more as needed
}
```

Replace all hard‑coded strings with `StripeKeys.*`.  
Extract amount conversion:

```java
private long toCents(BigDecimal amount, String currency) {
    int scale = Currency.getInstance(currency).getDefaultFractionDigits();
    BigDecimal scaled = amount.setScale(scale, RoundingMode.HALF_UP);
    return scaled.movePointRight(scale).longValueExact();
}
```

---

**Bottom line:**  
`Stripe3Payment` correctly orchestrates Stripe’s 3‑D Secure flow, but its implementation is heavily duplicated, uses a static API key setter that can cause race‑conditions, and mis‑records certain transaction attributes (especially for refunds). Refactoring the amount logic, centralising exception mapping, and replacing magic strings with constants will make the code far easier to test, maintain, and extend.

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
import com.stripe.exception.AuthenticationException;
import com.stripe.exception.CardException;
import com.stripe.exception.InvalidRequestException;
import com.stripe.exception.StripeException;
import com.stripe.model.PaymentIntent;
import com.stripe.model.Refund;
import com.stripe.param.PaymentIntentCaptureParams;
import com.stripe.param.PaymentIntentCreateParams;

// import com.stripe.exception.APIConnectionException;

public class Stripe3Payment implements PaymentModule {

	private static final Logger LOGGER = LoggerFactory.getLogger(Stripe3Payment.class);

	@Inject
	private ProductPriceUtils productPriceUtils;

	private final static String AUTHORIZATION = "Authorization";
	private final static String TRANSACTION = "Transaction";


	@Override
	public Transaction initTransaction(MerchantStore store, Customer customer,
									   BigDecimal amount, Payment payment,
									   IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {


		String strAmount = String.valueOf(amount);
		strAmount = strAmount.replace(".","");

		Transaction transaction = new Transaction();
		try {


			String apiKey = configuration.getIntegrationKeys().get("secretKey");

			if (StringUtils.isBlank(apiKey)) {
				IntegrationException te = new IntegrationException(
						"Can't process Stripe, missing payment.metaData");
				te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
				te.setMessageCode("message.payment.error");
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				throw te;
			}

			Stripe.apiKey = apiKey;

			PaymentIntentCreateParams createParams = new PaymentIntentCreateParams.Builder()
					.setCurrency(store.getCurrency().getCode())
					.setAmount(Long.parseLong(strAmount))
					.setCaptureMethod(PaymentIntentCreateParams.CaptureMethod.MANUAL)
					.build();

			// Create a PaymentIntent with the order amount and currency
			PaymentIntent intent = PaymentIntent.create(createParams);

			intent.getClientSecret();

			transaction.setAmount(amount);
			//transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.AUTHORIZE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", intent.getId());
			transaction.getTransactionDetails().put("TRNAPPROVED", intent.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", intent.getId());
			transaction.getTransactionDetails().put("INTENTSECRET", intent.getClientSecret());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);

		} catch (Exception e) {
			throw buildException(e);
		}

		return transaction;
	}

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


	/* -----------------------------------------  */

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
			 * This is the PaymentIntent ID created on the Frontend, that we will now store.
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

			Stripe.apiKey = apiKey;
			
			PaymentIntent paymentIntent = PaymentIntent.retrieve(
					token
			);
			
			transaction.setAmount(amount);
			//transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.AUTHORIZE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", token);
			transaction.getTransactionDetails().put("TRNAPPROVED", paymentIntent.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", paymentIntent.getId());  // <---- We store the PI id here
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

			String chargeId = capturableTransaction.getTransactionDetails().get("TRNORDERNUMBER");       // <---- We retrieve the PI id here

			if(StringUtils.isBlank(chargeId)) {
				IntegrationException te = new IntegrationException(
						"Can't process Stripe capture, missing TRNORDERNUMBER");
				te.setExceptionType(IntegrationException.TRANSACTION_EXCEPTION);
				te.setMessageCode("message.payment.error");
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				throw te;
			}

			String amnt = productPriceUtils.getAdminFormatedAmount(store, order.getTotal());
			String strAmount = String.valueOf(amnt);
			strAmount = strAmount.replace(".","");

			Stripe.apiKey = apiKey;


			PaymentIntent paymentIntent = PaymentIntent.retrieve(
					chargeId
			);

			PaymentIntentCaptureParams params =
					PaymentIntentCaptureParams.builder()
							.setAmountToCapture(Long.parseLong(strAmount))
							.setStatementDescriptor(
									store.getStorename().length() > 22 ?
											store.getStorename().substring(0, 22) :
											store.getStorename()
							)
					.build();

			paymentIntent = paymentIntent.capture(params);

			transaction.setAmount(order.getTotal());
			transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.CAPTURE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", capturableTransaction.getTransactionDetails().get("TRANSACTIONID"));
			transaction.getTransactionDetails().put("TRNAPPROVED", paymentIntent.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", paymentIntent.getId());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);

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
		  token = payment.getPaymentMetaData().get("paymentToken");
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
			
			/*Map<String, Object> chargeParams = new HashMap<String, Object>();
			chargeParams.put("amount", strAmount);
			chargeParams.put("capture", true);
			chargeParams.put("currency", store.getCurrency().getCode());
			chargeParams.put("source", token); // obtained with Stripe.js
			chargeParams.put("description", new StringBuilder().append(TRANSACTION).append(" - ").append(store.getStorename()).toString());
			*/

			Stripe.apiKey = apiKey;
			

			PaymentIntent paymentIntent = PaymentIntent.retrieve(
					token
		  	);

			PaymentIntentCaptureParams params =
					PaymentIntentCaptureParams.builder()
							.setAmountToCapture(Long.parseLong(strAmount))
							.setStatementDescriptor(
									store.getStorename().length() > 22 ?
											store.getStorename().substring(0, 22) :
											store.getStorename()
							)
					.build();

			paymentIntent = paymentIntent.capture(params);
	
			//Map<String,String> metadata = ch.getMetadata();
			
			
			transaction.setAmount(amount);
			//transaction.setOrder(order);
			transaction.setTransactionDate(new Date());
			transaction.setTransactionType(TransactionType.AUTHORIZECAPTURE);
			transaction.setPaymentType(PaymentType.CREDITCARD);
			transaction.getTransactionDetails().put("TRANSACTIONID", token);
			transaction.getTransactionDetails().put("TRNAPPROVED", paymentIntent.getStatus());
			transaction.getTransactionDetails().put("TRNORDERNUMBER", paymentIntent.getId());
			transaction.getTransactionDetails().put("MESSAGETEXT", null);
			
		} catch (Exception e) {

			e.printStackTrace();

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

			PaymentIntent paymentIntent = PaymentIntent.retrieve(
					trnID
			);

			Map<String, Object> params = new HashMap<>();
			params.put("payment_intent", paymentIntent.getId());
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
		
	} /*else if (ex instanceof APIConnectionException) {
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
