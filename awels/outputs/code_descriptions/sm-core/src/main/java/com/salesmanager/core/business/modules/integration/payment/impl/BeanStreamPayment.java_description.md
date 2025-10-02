# BeanStreamPayment.java

## Review

## 1. Summary

**Purpose**  
`BeanStreamPayment` implements the `PaymentModule` interface to integrate the BeanStream payment gateway into the SalesManager application. It supports the full payment lifecycle – authorization, capture, refund, and a combined authorize‑and‑capture operation – all performed via HTTP POST calls to BeanStream’s backend API.

**Key Components**

| Class/Interface | Role |
|-----------------|------|
| `BeanStreamPayment` | Concrete payment module implementation. Handles request construction, HTTP communication, response parsing, and transaction persistence. |
| `ProductPriceUtils` | Utility to format amounts in the currency format expected by BeanStream. |
| `MerchantLogService` | Persists logs for declined or error responses. |
| `IntegrationConfiguration` | Holds credentials and environment (TEST/PROD) settings. |
| `IntegrationModule` | Stores module‑specific configurations (scheme, host, port, URI). |
| `Transaction`, `Payment`, `Order`, `Customer` | Domain models used to pass data to and from the gateway. |

**Design Notes**

* The class follows a procedural style with heavy duplication across the different transaction types.
* Logging is performed via SLF4J, and error handling is unified through the custom `IntegrationException`.
* The code heavily relies on string concatenation to build URL‑encoded request bodies rather than using a library or helper method, which can lead to bugs if special characters are not encoded correctly.
* The implementation is tightly coupled to BeanStream’s specific API (e.g., transaction types “P”, “PA”, “PAC”, “R”) and assumes specific response keys (`TRNAPPROVED`, `TRNID`, etc.).

---

## 2. Detailed Description

### Execution Flow

1. **Incoming Request**  
   - The caller invokes one of the public `PaymentModule` methods (`authorize`, `capture`, `refund`, etc.).  
   - Parameters such as `MerchantStore`, `Customer`, `Payment`, `Order`, and `IntegrationConfiguration` are supplied.

2. **Preparation**  
   - For each operation, a request string is built manually by concatenating key/value pairs, often using a `StringBuilder`.  
   - The request includes authentication credentials (`merchant_id`, `username`, `password`) and transaction data (amount, card details, order info).

3. **HTTP Communication**  
   - `sendTransaction` is the central method that opens an `HttpURLConnection`, writes the URL‑encoded body, and reads the response.  
   - Response parsing uses `formatUrlResponse`, turning the raw response into a `Map<String,String>` of key/value pairs.

4. **Response Handling**  
   - The response is inspected for the `TRNAPPROVED` flag. If the transaction was declined (`0`), a `MerchantLog` is created and an `IntegrationException` with `EXCEPTION_PAYMENT_DECLINED` is thrown.  
   - For approved responses, `parseResponse` constructs a `Transaction` object containing the transaction ID, approval status, order number, and message text.

5. **Return**  
   - The constructed `Transaction` (or `null` in the unimplemented `initTransaction`) is returned to the caller.

6. **Cleanup**  
   - All streams and connections are closed in finally blocks to avoid resource leaks.

### Key Assumptions & Constraints

| Assumption | Effect |
|------------|--------|
| All credentials are present and non‑blank | Missing keys cause an `IntegrationException` in `validateModuleConfiguration`. |
| The `Configuration` environment string is either `"TEST"` or `"PROD"` | Determines which module config is used. |
| BeanStream’s API expects `application/x-www-form-urlencoded` bodies and returns URL‑encoded key/value pairs | Response parsing is simplistic and may break if the format changes. |
| Amounts are formatted via `ProductPriceUtils.getAdminFormatedAmount` | Expects the gateway to accept string representations with two decimal places. |
| Transaction type codes (“P”, “PA”, “PAC”, “R”) are hard‑coded | No flexibility for future gateway changes. |

### Architecture & Design Choices

* **Single Class Implementation** – All logic is encapsulated in one large class, which makes the code difficult to test in isolation.
* **Manual String Handling** – The code constructs request bodies manually rather than using a form helper or a dedicated HTTP client library (e.g., Apache HttpClient or Spring’s RestTemplate). This reduces maintainability.
* **Centralized HTTP Method (`sendTransaction`)** – Although this reduces duplication, the method still contains code that is specific to the request format and response handling for all transaction types.
* **Error Logging** – Errors are logged via `MerchantLogService` but there is no correlation ID or request tracing.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `initTransaction` | Unimplemented placeholder. | `store, customer, amount, payment, configuration, module` | `Transaction` (currently `null`) | None |
| `authorize` | Starts an authorization-only transaction. | Same as above | `Transaction` | Calls `processTransaction` |
| `capture` | Completes a pre‑authorization (PAC). | `store, customer, order, capturableTransaction, configuration, module` | `Transaction` | Builds request, calls `sendTransaction` |
| `authorizeAndCapture` | Performs a combined auth‑capture. | Same as `authorize` | `Transaction` | Calls `processTransaction` |
| `refund` | Processes a refund. | `partial, store, transaction, order, amount, configuration, module` | `Transaction` | Builds request, calls `sendTransaction` |
| `sendTransaction` | Generic helper to POST to BeanStream, read response, parse it. | `orderNumber, store, transaction, beanstreamType, transactionType, paymentType, amount, configuration, module` | `Transaction` | Performs HTTP I/O, logs declined responses |
| `processTransaction` | Builds request for purchase / pre‑auth. | `store, customer, type, amount, payment, configuration, module` | `Transaction` | Builds request, calls `sendTransaction` |
| `parseResponse` | Converts NVP map into a `Transaction` object. | `transactionType, paymentType, nvp, amount` | `Transaction` | None |
| `formatUrlResponse` | Parses URL‑encoded key/value string into a `Map`. | `payload` | `Map<String,String>` | None |
| `validateModuleConfiguration` | Validates that merchant credentials are present. | `integrationConfiguration, store` | None (throws on error) | Throws `IntegrationException` if missing keys |

**Reusable / Utility Methods**

* `formatUrlResponse` – a simple NVP parser.
* `parseResponse` – centralizes transaction object creation.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.inject.Inject` | **Framework** | Standard CDI injection; used for `ProductPriceUtils` and `MerchantLogService`. |
| `org.apache.commons.lang3.StringUtils` | **Utility** | Used for null/blank checks. |
| `org.slf4j.Logger` / `org.slf4j.LoggerFactory` | **Logging** | Standard SLF4J abstraction. |
| `salesmanager.core.utils.ProductPriceUtils` | **Internal** | Formats monetary amounts. |
| `salesmanager.core.service.MerchantLogService` | **Internal** | Persists logs for declined or error responses. |
| `salesmanager.integration.config.IntegrationConfiguration` | **Internal** | Holds credentials and environment. |
| `salesmanager.integration.config.IntegrationModule` | **Internal** | Holds gateway connection details. |
| `salesmanager.core.domain.*` (`Transaction`, `Payment`, `Order`, `Customer`, etc.) | **Internal** | Domain models. |
| `java.net.HttpURLConnection` | **Standard Library** | For HTTP POST/GET. |
| `java.net.URLDecoder` | **Standard Library** | Decodes NVP payload. |
| `java.util.*` | **Standard Library** | Collections, tokenizers. |
| `java.util.Date` | **Standard Library** | For transaction timestamp. |
| `java.util.StringTokenizer` | **Standard Library** | Tokenizes response string. |

No external HTTP client libraries are used, and all code is written against Java SE/EE core APIs.

---

## 5. Additional Notes / Recommendations

### 5.1. Code Duplication & Maintainability
* `processTransaction` and `capture` build very similar request strings; most fields are repeated.
* The hard‑coded transaction type codes and parameter names should be extracted into constants or an enum for better clarity and future change‑ability.
* The method `processTransaction` even contains a debug block that uses a different request string (`messageLogString`) that is never sent, creating confusion for developers.

### 5.2. String Handling & Encoding
* Requests are built by concatenating raw values (e.g., card numbers, names, addresses) without URL‑encoding. If any value contains a `&` or `=` character, the request will break.  
  * **Fix:** Use `URLEncoder.encode(value, "UTF-8")` for every value or, better yet, use a form‑builder or HTTP client that handles this automatically.
* In the refund request, the `orderNumber` parameter is sent as `trnOrderNumber`, but the comment shows the expected key is `trnOrderNumber`. Consistency would help readability.

### 5.3. Error Handling & Logging
* Declined responses are logged via `MerchantLogService`, but the log entry contains only the raw message text. Adding a correlation ID or request trace would make debugging easier.
* The `sendTransaction` method throws a generic `Exception` when parsing fails, but the calling methods wrap it in an `IntegrationException`. It would be cleaner to let `formatUrlResponse` throw `IntegrationException` directly.

### 5.4. Resource Management
* Streams are closed in `finally` blocks, but the code still calls `conn.disconnect()` inside `finally` even when `conn` is `null`. While harmless, it can be omitted or guarded more clearly.

### 5.5. Security
* Credentials are stored in plain text in `IntegrationConfiguration`. Ensure that the configuration repository encrypts these values in production.
* Card details are transmitted over plain HTTP unless the module config specifies HTTPS. The code does not enforce SSL/TLS usage, which is a potential security risk.

### 5.6. Extensibility
* The class is not designed to be easily extended for new payment gateways. A more flexible approach would involve:
  * Abstracting the request‑/response format into separate strategy classes.
  * Using dependency‑injected HTTP clients.
  * Exposing a generic `HttpRequestBuilder` utility.

### 5.7. Unit Testability
* Most public methods delegate to private methods that perform I/O. For unit testing, consider injecting a wrapper around `HttpURLConnection` or using a mocking framework (e.g., WireMock) to simulate the gateway.

---

### Overall Verdict

`BeanStreamPayment` fulfills its primary responsibilities but suffers from code duplication, brittle string handling, and tight coupling to the BeanStream API. Refactoring into smaller, reusable components (e.g., a `BeanStreamClient`, `BeanStreamRequestBuilder`, `BeanStreamResponseParser`) would greatly improve readability, testability, and future maintenance. The current implementation works for the present API but is fragile against changes in the gateway’s request/response format or in the surrounding application infrastructure.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.payment.impl;

import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.InputStreamReader;
import java.math.BigDecimal;
import java.net.HttpURLConnection;
import java.net.URL;
import java.net.URLDecoder;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;
import java.util.UUID;

import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.services.system.MerchantLogService;
import com.salesmanager.core.business.utils.CreditCardUtils;
import com.salesmanager.core.business.utils.ProductPriceUtils;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.payments.CreditCardPayment;
import com.salesmanager.core.model.payments.Payment;
import com.salesmanager.core.model.payments.PaymentType;
import com.salesmanager.core.model.payments.Transaction;
import com.salesmanager.core.model.payments.TransactionType;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.model.system.MerchantLog;
import com.salesmanager.core.model.system.ModuleConfig;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.payment.model.PaymentModule;


public class BeanStreamPayment implements PaymentModule {
	
	@Inject
	private ProductPriceUtils productPriceUtils;
	
	@Inject
	private MerchantLogService merchantLogService;
	

	
	private static final Logger LOGGER = LoggerFactory.getLogger(BeanStreamPayment.class);

	@Override
	public Transaction initTransaction(MerchantStore store, Customer customer,
			BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {
		// TODO Auto-generated method stub
		return null;
	}

	@Override
	public Transaction authorize(MerchantStore store, Customer customer,
			List<ShoppingCartItem> items, BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {
		return processTransaction(store, customer, TransactionType.AUTHORIZE,
				amount,
				payment,
				configuration,
				module);
	}

	@Override
	public Transaction capture(MerchantStore store, Customer customer,
			Order order, Transaction capturableTransaction,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {


			try {
				

				
				//authorize a preauth 

		
				String trnID = capturableTransaction.getTransactionDetails().get("TRANSACTIONID");
				
				String amnt = productPriceUtils.getAdminFormatedAmount(store, order.getTotal());
				
				/**
				merchant_id=123456789&requestType=BACKEND
				&trnType=PAC&username=user1234&password=pass1234&trnID=1000
				2115 --> requires also adjId [not documented]
				**/
				
				StringBuilder messageString = new StringBuilder();
				messageString.append("requestType=BACKEND&");
				messageString.append("merchant_id=").append(configuration.getIntegrationKeys().get("merchantid")).append("&");
				messageString.append("trnType=").append("PAC").append("&");
				messageString.append("username=").append(configuration.getIntegrationKeys().get("username")).append("&");
				messageString.append("password=").append(configuration.getIntegrationKeys().get("password")).append("&");
				messageString.append("trnAmount=").append(amnt).append("&");
				messageString.append("adjId=").append(trnID).append("&");
				messageString.append("trnID=").append(trnID);
				
				LOGGER.debug("REQUEST SENT TO BEANSTREAM -> " + messageString.toString());



				return sendTransaction(null, store, messageString.toString(), "PAC", TransactionType.CAPTURE, PaymentType.CREDITCARD, order.getTotal(), configuration, module);
				
			} catch(Exception e) {
				
				if(e instanceof IntegrationException)
					throw (IntegrationException)e;
				throw new IntegrationException("Error while processing BeanStream transaction",e);
	
			} 

	}

	@Override
	public Transaction authorizeAndCapture(MerchantStore store, Customer customer,
			List<ShoppingCartItem> items, BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {
		return processTransaction(
				store,
				customer,
				TransactionType.AUTHORIZECAPTURE,
				amount,
				payment,
				configuration,
				module);
	}

	@Override
	public Transaction refund(boolean partial, MerchantStore store, Transaction transaction,
			Order order, BigDecimal amount,
			IntegrationConfiguration configuration, IntegrationModule module)
			throws IntegrationException {

		
		
		
		HttpURLConnection conn = null;
		
		try {
			
			
			boolean bSandbox = false;
			if (configuration.getEnvironment().equals("TEST")) {// sandbox
				bSandbox = true;
			}

			ModuleConfig configs = module.getModuleConfigs().get("PROD");

			if (bSandbox) {
				configs = module.getModuleConfigs().get("TEST");
			} 
			
			if(configs==null) {
				throw new IntegrationException("Module not configured for TEST or PROD");
			}


			String server = new StringBuffer().append(
					
					configs.getScheme()).append("://")
					.append(configs.getHost())
							.append(":")
							.append(configs.getPort())
							.append(configs.getUri()).toString();

			String trnID = transaction.getTransactionDetails().get("TRANSACTIONID");
			
			String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
			
			/**
			merchant_id=123456789&requestType=BACKEND
			&trnType=R&username=user1234&password=pass1234
			&trnOrderNumber=1234&trnAmount=1.00&adjId=1000
			2115
			**/
			StringBuilder messageString = new StringBuilder();



			messageString.append("requestType=BACKEND&");
			messageString.append("merchant_id=").append(configuration.getIntegrationKeys().get("merchantid")).append("&");
			messageString.append("trnType=").append("R").append("&");
			messageString.append("username=").append(configuration.getIntegrationKeys().get("username")).append("&");
			messageString.append("password=").append(configuration.getIntegrationKeys().get("password")).append("&");
			messageString.append("trnOrderNumber=").append(transaction.getTransactionDetails().get("TRNORDERNUMBER")).append("&");
			messageString.append("trnAmount=").append(amnt).append("&");
			messageString.append("adjId=").append(trnID);
			
			LOGGER.debug("REQUEST SENT TO BEANSTREAM -> " + messageString.toString());
	
			
		
			
			URL postURL = new URL(server);
			conn = (HttpURLConnection) postURL.openConnection();




			return sendTransaction(null, store, messageString.toString(), "R", TransactionType.REFUND, PaymentType.CREDITCARD, amount, configuration, module);
			
		} catch(Exception e) {
			
			if(e instanceof IntegrationException)
				throw (IntegrationException)e;
			throw new IntegrationException("Error while processing BeanStream transaction",e);

		} finally {
			
			
			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}
		}
		
		
		
	}
	
	
	private Transaction sendTransaction(
			String orderNumber,
			MerchantStore store,
			String transaction, 
			String beanstreamType, 
			TransactionType transactionType,
			PaymentType paymentType,
			BigDecimal amount,
			IntegrationConfiguration configuration,
			IntegrationModule module
			) throws IntegrationException {
		
		String agent = "Mozilla/4.0";
		String respText = "";
		Map<String,String> nvp = null;
		DataOutputStream output = null;
		DataInputStream in = null;
		BufferedReader is = null;
		HttpURLConnection conn =null;
		try {
			
			//transaction = "requestType=BACKEND&merchant_id=300200260&trnType=P&username=carlito&password=shopizer001&orderNumber=caa71106-7e3f-4975-a657-a35904dc32a0&trnCardOwner=Carl Samson&trnCardNumber=5100000020002000&trnExpMonth=10&trnExpYear=14&trnCardCvd=123&trnAmount=77.01&ordName=Carl S&ordAddress1=358 Du Languedoc&ordCity=Victoria&ordProvince=BC&ordPostalCode=V8T2E7&ordCountry=CA&ordPhoneNumber=(444) 555-6666&ordEmailAddress=csamson777@yahoo.com";
			/**
			requestType=BACKEND&merchant_id=300200260
			&trnType=P
			&username=carlito&password=shopizer001
			&orderNumber=caa71106-7e3f-4975-a657-a35904dc32a0
			&trnCardOwner=Carl Samson
			&trnCardNumber=5100000020002000
			&trnExpMonth=10
			&trnExpYear=14
			&trnCardCvd=123
			&trnAmount=77.01
			&ordName=Carl S
			&ordAddress1=378 Du Languedoc
			&ordCity=Boucherville
			&ordProvince=QC
			&ordPostalCode=J3B8J1
			&ordCountry=CA
			&ordPhoneNumber=(444) 555-6666
			&ordEmailAddress=test@yahoo.com
			**/
			
			/**
			merchant_id=123456789&requestType=BACKEND
			&trnType=P&trnOrderNumber=1234TEST&trnAmount=5.00&trnCardOwner=Joe+Test
					&trnCardNumber=4030000010001234
					&trnExpMonth=10
					&trnExpYear=16
					&ordName=Joe+Test
					&ordAddress1=123+Test+Street
					&ordCity=Victoria
					&ordProvince=BC
					&ordCountry=CA
					&ordPostalCode=V8T2E7
					&ordPhoneNumber=5555555555
					&ordEmailAddress=joe%40testemail.com
			**/
			
			
			
			boolean bSandbox = false;
			if (configuration.getEnvironment().equals("TEST")) {// sandbox
				bSandbox = true;
			}

			String server = "";
			
			ModuleConfig configs = module.getModuleConfigs().get("PROD");

			if (bSandbox) {
				configs = module.getModuleConfigs().get("TEST");
			} 
			
			if(configs==null) {
				throw new IntegrationException("Module not configured for TEST or PROD");
			}
			

			server = new StringBuffer().append(
					
					configs.getScheme()).append("://")
					.append(configs.getHost())
							.append(":")
							.append(configs.getPort())
							.append(configs.getUri()).toString();
			
	
			
			URL postURL = new URL(server);
			conn = (HttpURLConnection) postURL.openConnection();
			

			// Set connection parameters. We need to perform input and output,
			// so set both as true.
			conn.setDoInput(true);
			conn.setDoOutput(true);

			// Set the content type we are POSTing. We impersonate it as
			// encoded form data
			conn.setRequestProperty("Content-Type",
					"application/x-www-form-urlencoded");
			conn.setRequestProperty("User-Agent", agent);

			conn.setRequestProperty("Content-Length", String
					.valueOf(transaction.length()));
			conn.setRequestMethod("POST");

			// get the output stream to POST to.
			output = new DataOutputStream(conn.getOutputStream());
			output.writeBytes(transaction);
			output.flush();


			// Read input from the input stream.
			in = new DataInputStream(conn.getInputStream());
			int rc = conn.getResponseCode();
			if (rc != -1) {
				is = new BufferedReader(new InputStreamReader(conn
						.getInputStream()));
				String _line = null;
				while (((_line = is.readLine()) != null)) {
					respText = respText + _line;
				}
				
				LOGGER.debug("BeanStream response -> " + respText.trim());
				
				nvp = formatUrlResponse(respText.trim());
			} else {
				throw new IntegrationException("Invalid response from BeanStream, return code is " + rc);
			}
			
			//check
			//trnApproved=1&trnId=10003067&messageId=1&messageText=Approved&trnOrderNumber=E40089&authCode=TEST&errorType=N&errorFields=

			String transactionApproved = nvp.get("TRNAPPROVED");
			String transactionId = nvp.get("TRNID");
			String messageId = nvp.get("MESSAGEID");
			String messageText = (String)nvp.get("MESSAGETEXT");
			String orderId = (String)nvp.get("TRNORDERNUMBER");
			String authCode = (String)nvp.get("AUTHCODE");
			String errorType = (String)nvp.get("ERRORTYPE");
			String errorFields = (String)nvp.get("ERRORFIELDS");
			if(!StringUtils.isBlank(orderNumber)) {
				nvp.put("INTERNALORDERID", orderNumber);
			}
			
			if(StringUtils.isBlank(transactionApproved)) {
				throw new IntegrationException("Required field transactionApproved missing from BeanStream response");
			}
			
			//errors
			if(transactionApproved.equals("0")) {

				merchantLogService.save(
						new MerchantLog(store,
						"Can't process BeanStream message "
								 + messageText + " return code id " + messageId));
	
				IntegrationException te = new IntegrationException(
						"Can't process BeanStream message " + messageText);
				te.setExceptionType(IntegrationException.EXCEPTION_PAYMENT_DECLINED);
				te.setMessageCode("message.payment.beanstream." + messageId);
				te.setErrorCode(IntegrationException.TRANSACTION_EXCEPTION);
				throw te;
			}
			
			//create transaction object

			//return parseResponse(type,transaction,respText,nvp,order);
			return this.parseResponse(transactionType, paymentType, nvp, amount);
			
			
		} catch(Exception e) {
			if(e instanceof IntegrationException) {
				throw (IntegrationException)e;
			}
			
			throw new IntegrationException("Error while processing BeanStream transaction",e);

		} finally {
			if (is != null) {
				try {
					is.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (in != null) {
				try {
					in.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (output != null) {
				try {
					output.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}
			
			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

		}

		
	}
	
	
	
	private Transaction processTransaction(MerchantStore store, Customer customer, TransactionType type,
			BigDecimal amount, Payment payment,
			IntegrationConfiguration configuration, IntegrationModule module) throws IntegrationException {
		

		
		
		
		boolean bSandbox = false;
		if (configuration.getEnvironment().equals("TEST")) {// sandbox
			bSandbox = true;
		}

		ModuleConfig configs = module.getModuleConfigs().get("PROD");

		if (bSandbox) {
			configs = module.getModuleConfigs().get("TEST");
		} 
		
		if(configs==null) {
			throw new IntegrationException("Module not configured for TEST or PROD");
		}


		String server = new StringBuffer().append(
				
				configs.getScheme()).append("://")
				.append(configs.getHost())
						.append(":")
						.append(configs.getPort())
						.append(configs.getUri()).toString();
		
		HttpURLConnection conn = null;
		
		try {
			
		String uniqueId = UUID.randomUUID().toString();//TODO
			
		String orderNumber = uniqueId;
		
		String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
		
		
		StringBuilder messageString = new StringBuilder();
		
		String transactionType = "P";
		if(type == TransactionType.AUTHORIZE) {
			transactionType = "PA";
		} else if(type == TransactionType.AUTHORIZECAPTURE) {
			transactionType = "P";
		} 
		
		CreditCardPayment creditCardPayment = (CreditCardPayment)payment;

		messageString.append("requestType=BACKEND&");
		messageString.append("merchant_id=").append(configuration.getIntegrationKeys().get("merchantid")).append("&");
		messageString.append("trnType=").append(transactionType).append("&");
		messageString.append("username=").append(configuration.getIntegrationKeys().get("username")).append("&");
		messageString.append("password=").append(configuration.getIntegrationKeys().get("password")).append("&");
		messageString.append("orderNumber=").append(orderNumber).append("&");
		messageString.append("trnCardOwner=").append(creditCardPayment.getCardOwner()).append("&");
		messageString.append("trnCardNumber=").append(creditCardPayment.getCreditCardNumber()).append("&");
		messageString.append("trnExpMonth=").append(creditCardPayment.getExpirationMonth()).append("&");
		messageString.append("trnExpYear=").append(creditCardPayment.getExpirationYear().substring(2)).append("&");
		messageString.append("trnCardCvd=").append(creditCardPayment.getCredidCardValidationNumber()).append("&");
		messageString.append("trnAmount=").append(amnt).append("&");
		
		StringBuilder nm = new StringBuilder();
		nm.append(customer.getBilling().getFirstName()).append(" ").append(customer.getBilling().getLastName());
		
		
		messageString.append("ordName=").append(nm.toString()).append("&");
		messageString.append("ordAddress1=").append(customer.getBilling().getAddress()).append("&");
		messageString.append("ordCity=").append(customer.getBilling().getCity()).append("&");
		
		String stateProvince = customer.getBilling().getState();
		if(customer.getBilling().getZone()!=null) {
			stateProvince = customer.getBilling().getZone().getCode();
		}
		
		String countryName = customer.getBilling().getCountry().getIsoCode();
		
		messageString.append("ordProvince=").append(stateProvince).append("&");
		messageString.append("ordPostalCode=").append(customer.getBilling().getPostalCode().replaceAll("\\s","")).append("&");
		messageString.append("ordCountry=").append(countryName).append("&");
		messageString.append("ordPhoneNumber=").append(customer.getBilling().getTelephone()).append("&");
		messageString.append("ordEmailAddress=").append(customer.getEmailAddress());
		
		
		
		
		/**
		 * 	purchase (P)
		 *  -----------
				REQUEST -> merchant_id=123456789&requestType=BACKEND&trnType=P&trnOrderNumber=1234TEST&trnAmount=5.00&trnCardOwner=Joe+Test&trnCardNumber=4030000010001234&trnExpMonth=10&trnExpYear=10&ordName=Joe+Test&ordAddress1=123+Test+Street&ordCity=Victoria&ordProvince=BC&ordCountry=CA&ordPostalCode=V8T2E7&ordPhoneNumber=5555555555&ordEmailAddress=joe%40testemail.com
				RESPONSE-> trnApproved=1&trnId=10003067&messageId=1&messageText=Approved&trnOrderNumber=E40089&authCode=TEST&errorType=N&errorFields=&responseType=T&trnAmount=10%2E00&trnDate=1%2F17%2F2008+11%3A36%3A34+AM&avsProcessed=0&avsId=0&avsResult=0&avsAddrMatch=0&avsPostalMatch=0&avsMessage=Address+Verification+not+performed+for+this+transaction%2E&rspCodeCav=0&rspCavResult=0&rspCodeCredit1=0&rspCodeCredit2=0&rspCodeCredit3=0&rspCodeCredit4=0&rspCodeAddr1=0&rspCodeAddr2=0&rspCodeAddr3=0&rspCodeAddr4=0&rspCodeDob=0&rspCustomerDec=&trnType=P&paymentMethod=CC&ref1=&ref2=&ref3=&ref4=&ref5=
		
			pre authorization (PA)
			----------------------

			Prior to processing a pre-authorization through the API, you must modify the transaction settings in your Beanstream merchant member area to allow for this transaction type.
			- Log in to the Beanstream online member area at www.beanstream.com/admin/sDefault.asp.
			- Navigate to administration - account admin - order settings in the left menu.
			Under the heading �Restrict Internet Transaction Processing Types,� select either of the last two options. The �Purchases or Pre-Authorization Only� option will allow you to process both types of transaction through your web interface. De-selecting the �Restrict Internet Transaction Processing Types� checkbox will allow you to process all types of transactions including returns, voids and pre-auth completions.
		
			capture (PAC) -> requires trnId
			-------------
		
			refund (R)
			-------------
				REQUEST -> merchant_id=123456789&requestType=BACKEND&trnType=R&username=user1234&password=pass1234&trnOrderNumber=1234&trnAmount=1.00&adjId=10002115
				RESPONSE-> trnApproved=1&trnId=10002118&messageId=1&messageText=Approved&trnOrderNumber=1234R&authCode=TEST&errorType=N&errorFields=&responseType=T&trnAmount=1%2E00&trnDate=8%2F17%2F2009+1%3A44%3A56+PM&avsProcessed=0&avsId=0&avsResult=0&avsAddrMatch=0&avsPostalMatch=0&avsMessage=Address+Verification+not+performed+for+this+transaction%2E&cardType=VI&trnType=R&paymentMethod=CC&ref1=&ref2=&ref3=&ref4=&ref5=
		

			//notes
			//On receipt of the transaction response, the merchant must display order amount, transaction ID number, bank authorization code (authCode), currency, date and �messageText� to the customer on a confirmation page.
		*/
		

		//String agent = "Mozilla/4.0";
		//String respText = "";
		//Map nvp = null;
		
		
		/** debug **/
		
		

			StringBuilder messageLogString = new StringBuilder();
			
			
			messageLogString.append("requestType=BACKEND&");
			messageLogString.append("merchant_id=").append(configuration.getIntegrationKeys().get("merchantid")).append("&");
			messageLogString.append("trnType=").append(type).append("&");
			messageLogString.append("orderNumber=").append(orderNumber).append("&");
			messageLogString.append("trnCardOwner=").append(creditCardPayment.getCardOwner()).append("&");
			messageLogString.append("trnCardNumber=").append(CreditCardUtils.maskCardNumber(creditCardPayment.getCreditCardNumber())).append("&");
			messageLogString.append("trnExpMonth=").append(creditCardPayment.getExpirationMonth()).append("&");
			messageLogString.append("trnExpYear=").append(creditCardPayment.getExpirationYear()).append("&");
			messageLogString.append("trnCardCvd=").append(creditCardPayment.getCredidCardValidationNumber()).append("&");
			messageLogString.append("trnAmount=").append(amnt).append("&");

			messageLogString.append("ordName=").append(nm.toString()).append("&");
			messageLogString.append("ordAddress1=").append(customer.getBilling().getAddress()).append("&");
			messageLogString.append("ordCity=").append(customer.getBilling().getCity()).append("&");
			

			
			messageLogString.append("ordProvince=").append(stateProvince).append("&");
			messageLogString.append("ordPostalCode=").append(customer.getBilling().getPostalCode()).append("&");
			messageLogString.append("ordCountry=").append(customer.getBilling().getCountry().getName()).append("&");
			messageLogString.append("ordPhoneNumber=").append(customer.getBilling().getTelephone()).append("&");
			messageLogString.append("ordEmailAddress=").append(customer.getEmailAddress());
			
			


			/** debug **/
	
	
			LOGGER.debug("REQUEST SENT TO BEANSTREAM -> " + messageLogString.toString());

			
			URL postURL = new URL(server);
			conn = (HttpURLConnection) postURL.openConnection();



			return sendTransaction(orderNumber, store, messageString.toString(), transactionType, type, payment.getPaymentType(), amount, configuration, module);
			
		} catch(Exception e) {
			
			if(e instanceof IntegrationException)
				throw (IntegrationException)e;
			throw new IntegrationException("Error while processing BeanStream transaction",e);

		} finally {
			
			
			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {}
			}
		}

	}
	
	
	
	private Transaction parseResponse(TransactionType transactionType,
			PaymentType paymentType, Map<String,String> nvp,
			BigDecimal amount) throws Exception {
		
		
		Transaction transaction = new Transaction();
		transaction.setAmount(amount);
		//transaction.setOrder(order);
		transaction.setTransactionDate(new Date());
		transaction.setTransactionType(transactionType);
		transaction.setPaymentType(PaymentType.CREDITCARD);
		transaction.getTransactionDetails().put("TRANSACTIONID", (String)nvp.get("TRNID"));
		transaction.getTransactionDetails().put("TRNAPPROVED", (String)nvp.get("TRNAPPROVED"));
		transaction.getTransactionDetails().put("TRNORDERNUMBER", (String)nvp.get("TRNORDERNUMBER"));
		transaction.getTransactionDetails().put("MESSAGETEXT", (String)nvp.get("MESSAGETEXT"));
		if(nvp.get("INTERNALORDERID")!=null) {
			transaction.getTransactionDetails().put("INTERNALORDERID", (String)nvp.get("INTERNALORDERID"));
		}
		return transaction;
		
	}

	private Map<String,String> formatUrlResponse(String payload) throws Exception {
		HashMap<String,String> nvp = new HashMap<String,String> ();
		StringTokenizer stTok = new StringTokenizer(payload, "&");
		while (stTok.hasMoreTokens()) {
			StringTokenizer stInternalTokenizer = new StringTokenizer(stTok
					.nextToken(), "=");
			if (stInternalTokenizer.countTokens() == 2) {
				String key = URLDecoder.decode(stInternalTokenizer.nextToken(),
						"UTF-8");
				String value = URLDecoder.decode(stInternalTokenizer
						.nextToken(), "UTF-8");
				nvp.put(key.toUpperCase(), value);
			}
		}
		return nvp;
	}

	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		
		
		List<String> errorFields = null;
		
		
		Map<String,String> keys = integrationConfiguration.getIntegrationKeys();
		
		//validate integrationKeys['merchantid']
		if(keys==null || StringUtils.isBlank(keys.get("merchantid"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("merchantid");
		}
		
		//validate integrationKeys['username']
		if(keys==null || StringUtils.isBlank(keys.get("username"))) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("username");
		}
		
		
		//validate integrationKeys['password']
		if(keys==null || StringUtils.isBlank(keys.get("password"))) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("password");
		}


		
		if(errorFields!=null) {
			IntegrationException ex = new IntegrationException(IntegrationException.ERROR_VALIDATION_SAVE);
			ex.setErrorFields(errorFields);
			throw ex;
			
		}
		
		
		
	}



}



```
