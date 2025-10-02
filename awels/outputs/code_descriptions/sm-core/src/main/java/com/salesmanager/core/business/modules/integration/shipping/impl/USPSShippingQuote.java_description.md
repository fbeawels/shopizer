# USPSShippingQuote.java

## Review

## 1. Summary

**Purpose & Functionality**  
The `USPSShippingQuote` class implements the `ShippingQuoteModule` interface to obtain shipping rates from the USPS Rate v3/International API. It:

1. Validates that the integration configuration contains the required keys (USPS account number) and options (packages, optionally services).
2. Builds an XML request based on the order’s shipping packages, destination, and store settings.
3. Sends the request to the USPS endpoint, parses the XML response, and returns a list of `ShippingOption` objects.

**Key Components**

| Class | Role |
|-------|------|
| `USPSShippingQuote` | Orchestrates validation, request construction, HTTP communication, and response parsing. |
| `USPSParsedElements` | POJO used by Apache Digester to capture parsed XML into a list of `ShippingOption` instances. |
| `ProductPriceUtils` / `CountryService` | Injected services used to format prices and retrieve country information. |

**Design Patterns & Libraries**

* **Builder/Factory** – XML is built as a string; a more robust builder or XML DOM library would be preferable.  
* **Template Method** – `ShippingQuoteModule` is an interface; the class implements its contract.  
* **Dependency Injection** – `@Inject` annotations provide the services.  
* **Apache HttpClient** – For HTTP communication.  
* **Apache Digester** – For XML unmarshalling.  
* **SLF4J** – For logging.

---

## 2. Detailed Description

### Flow of Execution

1. **`getShippingQuotes`**  
   * Checks for null inputs (packages, postal code).  
   * Ensures the store is based in the US – the hard‑coded check (`store.getCountry().getIsoCode().equals("US")`).  
   * Loads configuration (keys, options, environment, module host/port).  
   * Calculates total weight, dimensions, girth, and shipping size classification.  
   * Builds the XML request for either domestic or international rates.  
   * Executes the HTTP GET request using `CloseableHttpClient`.  
   * Parses the XML response with `Digester` into a `USPSParsedElements` instance.  
   * Transforms `ShippingOption` objects (currency conversion, description formatting) and returns them.

2. **`validateModuleConfiguration`** – Verifies that the integration keys and options are present and not empty, throwing `IntegrationException` on validation failure.

3. **`getCustomModuleConfiguration`** – Currently a no‑op; returns `null`.

### Key Assumptions & Constraints

* **US Only** – The store must be in the United States; otherwise an exception is thrown.  
* **Locale Support** – Only English and French are accepted; any other locale defaults to English.  
* **Package Options** – The first package type from configuration is used (`options.get("packages").get(0)`), ignoring any others.  
* **XML Encoding** – `URLEncoder.encode` (no charset specified) is used to encode the XML string.  
* **HTTP Method** – Uses `HttpGet` with query parameters instead of a POST body.  
* **Currency Handling** – For international rates, the order total is passed straight to `productPriceUtils.getAdminFormatedAmount`; no currency conversion logic is present.

### Architecture & Design Choices

* The class is **state‑less** (except for injected services), making it thread‑safe.  
* Using **StringBuilder** for XML and manual string concatenation introduces potential XML injection risks and makes the code hard to read.  
* The **Digester** approach is concise but tightly couples the XML structure to Java code.  
* Error handling is limited to `IntegrationException` and generic logging; no retry logic or HTTP status‑code mapping beyond 200–299.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `validateModuleConfiguration(IntegrationConfiguration, MerchantStore)` | Ensures required keys and options are present. | *integrationConfiguration*, *store* | void | Throws `IntegrationException` if validation fails. |
| `getShippingQuotes(ShippingQuote, List<PackageDetails>, BigDecimal, Delivery, ShippingOrigin, MerchantStore, IntegrationConfiguration, IntegrationModule, ShippingConfiguration, Locale)` | Builds request, calls USPS, parses response, returns shipping options. | Order & shipment details | `List<ShippingOption>` | Throws `IntegrationException` on error; logs debug info. |
| `getCustomModuleConfiguration(MerchantStore)` | Stub for custom configuration. | *store* | `CustomIntegrationConfiguration` (always `null`) | None. |
| `USPSParsedElements.addOption(ShippingOption)` | Adds parsed option to internal list. | *option* | void | Modifies internal list. |
| `USPSParsedElements.getOptions()` | Returns parsed options. | None | `List<ShippingOption>` | None. |
| `USPSParsedElements.getError()` | Retrieves error message from the parsed XML. | None | `String` | None. |

*Utility methods are embedded in the `DataUtils`, `ProductPriceUtils`, and `MeasureUnit` classes.*

---

## 4. Dependencies

| External Library | Purpose | Standard / Third‑Party |
|-------------------|---------|------------------------|
| `org.apache.commons.lang3.StringUtils` | String null/blank checks | Third‑Party |
| `org.apache.commons.digester.Digester` | XML parsing into POJOs | Third‑Party |
| `org.apache.http.client.*` (`HttpClient`, `HttpGet`, `EntityUtils`) | HTTP communication with USPS | Third‑Party |
| `org.slf4j.*` | Logging | Third‑Party |
| `javax.inject.Inject` | Dependency injection | Java EE (or CDI) |
| SalesManager core classes (`ShippingQuoteModule`, `ShippingOption`, etc.) | Domain model | Project‑Specific |

---

## 5. Strengths

1. **Clear Separation of Concerns** – Validation, request creation, HTTP call, and response parsing are logically separated.  
2. **Thread‑Safety** – No mutable instance state; services are injected.  
3. **Resource Management** – `CloseableHttpClient` is used within a try‑with‑resources block; connections are released in a finally clause.  
4. **Logging** – Uses SLF4J with debug and error levels, which aids troubleshooting.  
5. **Extensibility** – By implementing `ShippingQuoteModule` the code can be swapped with another carrier implementation.

---

## 5. Areas for Improvement & Recommendations

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded “US” store check** | Prevents using the module for other countries; reduces flexibility. | Make the allowed country configurable or extend the module to support other carriers. |
| **Locale handling** | Non‑supported locales silently default to English, which may be misleading. | Throw a validation error or document the defaulting behavior. |
| **XML building** | String concatenation is error‑prone and opens up injection possibilities. | Use an XML library (e.g., JDOM, JAXB) or a dedicated XML builder. |
| **First package only** | Ignores other package options that may be defined. | Accept all package types from configuration. |
| **URLEncoder usage** | Charset is unspecified; may lead to incorrect encoding on systems using non‑UTF‑8 defaults. | Use `URLEncoder.encode(String, StandardCharsets.UTF_8)` or `HttpPost` with a body. |
| **HTTP method** | GET with query string can hit URL length limits for large XML. | Switch to `HttpPost` and send the XML as the request body. |
| **Error handling** | Only `IntegrationException` is thrown; HTTP status codes other than 200–299 are not mapped. | Add a dedicated error mapper; optionally implement retry logic for transient failures. |
| **Currency conversion** | International rates always assume USD; no conversion to the store’s currency. | Use the same currency conversion logic as for domestic rates. |
| **Null checks** | `store.getCountry()` or `store.getCountry().getIsoCode()` may return `null`. | Add null checks before using these values. |
| **Raw List in USPSParsedElements** | Suppresses generics. | Change `List` to `List<ShippingOption>` and add `<T>` generic where appropriate. |
| **Duplicate and commented code** | Reduces readability and increases maintenance burden. | Remove dead/commented sections and extract helper methods (e.g., XML construction, dimension calculation). |
| **No retry logic** | Single‑attempt failure may lead to user‑visible errors. | Add a simple retry loop or delegate to a higher‑level retry strategy. |
| **No rate‑limit handling** | Frequent calls may hit USPS limits. | Respect `X-RateLimit-*` headers if available; throttle accordingly. |

---

## 5. Edge Cases & Robustness

| Edge Case | Current Behaviour | Suggested Behaviour |
|-----------|-------------------|---------------------|
| `packages` is `null` or empty | Returns `null`. | Return an empty list and log a warning. |
| `delivery.getPostalCode()` is `null` | `StringUtils.isBlank` will throw `NullPointerException`. | Check for `null` before `isBlank`. |
| `configuration` missing keys or options | Throws `IntegrationException` – fine. | None. |
| HTTPS endpoint mis‑configured or missing | `ClientProtocolException` thrown. | Map all HTTP error codes to meaningful `IntegrationException` messages. |
| XML response contains multiple `<Error>` nodes | Only the first error is captured. | Capture all errors and aggregate them for debugging. |
| Order total currency mismatch for international rates | No conversion. | Convert order total to the USPS expected currency (USD) before sending. |

---

## 6. Suggested Refactoring & Enhancements

1. **XML Generation**  
   * Replace string concatenation with an XML builder (e.g., `javax.xml.stream.XMLOutputFactory`) or JAXB to guarantee well‑formedness and protect against injection.  
   * Parameterize package type list rather than taking only the first element.

2. **HTTP Interaction**  
   * Use `HttpPost` with a request body containing the XML (recommended by USPS documentation).  
   * Specify UTF‑8 explicitly in `URLEncoder.encode` or skip encoding entirely if using POST.  
   * Handle non‑2xx status codes more gracefully (e.g., 400 = validation error, 500 = service unavailable).

3. **Currency Conversion**  
   * Extract a helper that converts the USPS USD rate to the store’s currency, reusing existing `CurrencyUtil` logic.  
   * For international rates, format `orderTotal` in the correct currency before passing to `productPriceUtils`.

4. **Unit Tests**  
   * Add tests for domestic and international scenarios, including error responses.  
   * Mock `CountryService` and `ProductPriceUtils` to verify unit conversions.

5. **Clean‑Up**  
   * Remove all commented‑out blocks; keep only the active logic.  
   * Add JavaDoc comments to public methods and the `USPSParsedElements` class.  
   * Use generics everywhere (e.g., `List<ShippingOption>` in `USPSParsedElements.getOptions()`).

6. **Exception Hierarchy**  
   * Consider a more granular exception hierarchy (`TransportException`, `ParseException`, `ValidationException`) instead of a single `IntegrationException`.

---

## 5. Overall Assessment

The implementation correctly orchestrates a USPS shipping quote request and response cycle, but it suffers from:

* **Hard‑coded assumptions** (US store only, first package type, fixed locales).  
* **Fragile XML handling** (manual string concatenation, potential injection).  
* **Limited error & retry handling**.  
* **Code clutter** (many unused variables and commented blocks).  
* **Minor API mis‑uses** (`URLEncoder.encode` without charset, use of `HttpGet` for a body payload).

With the suggested refactoring—cleaner XML construction, robust error mapping, and more flexible configuration handling—the module would be easier to maintain, more secure, and easier to extend to other carriers or locales.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.io.IOException;
import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.inject.Inject;

import org.apache.commons.digester.Digester;
import org.apache.commons.lang3.StringUtils;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.ClientProtocolException;
import org.apache.http.client.ResponseHandler;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.utils.DataUtils;
import com.salesmanager.core.business.utils.ProductPriceUtils;
import com.salesmanager.core.constants.MeasureUnit;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingOption;
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.system.CustomIntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.model.system.ModuleConfig;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuoteModule;


/**
 * Integrates with USPS online API
 * @author casams1
 *
 */
public class USPSShippingQuote implements ShippingQuoteModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(USPSShippingQuote.class);

	
	@Inject
	private ProductPriceUtils productPriceUtils;
	
	@Inject
	private CountryService countryService;
	

	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		
		
		List<String> errorFields = null;
		
		//validate integrationKeys['account']
		Map<String,String> keys = integrationConfiguration.getIntegrationKeys();
		if(keys==null || StringUtils.isBlank(keys.get("account"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("identifier");
		}

		//validate at least one integrationOptions['packages']
		Map<String,List<String>> options = integrationConfiguration.getIntegrationOptions();
		if(options==null) {
			errorFields = new ArrayList<String>();
			errorFields.add("identifier");
		}
		
		List<String> packages = options.get("packages");
		if(packages==null || packages.size()==0) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("packages");
		}
		
/*		List<String> services = options.get("services");
		if(services==null || services.size()==0) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("services");
		}
		
		if(services!=null && services.size()>3) {
			if(errorFields==null) {
				errorFields = new ArrayList<String>();
			}
			errorFields.add("services");
		}*/
		
		if(errorFields!=null) {
			IntegrationException ex = new IntegrationException(IntegrationException.ERROR_VALIDATION_SAVE);
			ex.setErrorFields(errorFields);
			throw ex;
			
		}
		
		

	}

	@Override
	public List<ShippingOption> getShippingQuotes(
			ShippingQuote shippingQuote,
			List<PackageDetails> packages, BigDecimal orderTotal,
			Delivery delivery, ShippingOrigin origin, MerchantStore store,
			IntegrationConfiguration configuration, IntegrationModule module,
			ShippingConfiguration shippingConfiguration, Locale locale)
			throws IntegrationException {

	

		if (packages == null) {
			return null;
		}
		
		if(StringUtils.isBlank(delivery.getPostalCode())) {
			return null;
		}
		


		// only applies to Canada and US
/*		Country country = delivery.getCountry();
		if(!country.getIsoCode().equals("US") || !country.getIsoCode().equals("US")){
			throw new IntegrationException("USPS Not configured for shipping in country " + country.getIsoCode());
		}*/
		


		// supports en and fr
		String language = locale.getLanguage();
		if (!language.equals(Locale.FRENCH.getLanguage())
				&& !language.equals(Locale.ENGLISH.getLanguage())) {
			language = Locale.ENGLISH.getLanguage();
		}
		

		// if store is not CAD /** maintained in the currency **/
/*		if (!store.getCurrency().equals(Constants.CURRENCY_CODE_CAD)) {
			total = CurrencyUtil.convertToCurrency(total, store.getCurrency(),
					Constants.CURRENCY_CODE_CAD);
		}*/
		
		Language lang = store.getDefaultLanguage();
		

		
		HttpGet  httpget = null;
		Reader xmlreader = null;
		String pack = configuration.getIntegrationOptions().get("packages").get(0);

		try {
			
			Map<String,Country> countries = countryService.getCountriesMap(lang);

			Country destination = countries.get(delivery.getCountry().getIsoCode());
			
		
			
			Map<String,String> keys = configuration.getIntegrationKeys();
			if(keys==null || StringUtils.isBlank(keys.get("account"))) {
				return null;//TODO can we return null
			}

			
			String host = null;
			String protocol = null;
			String port = null;
			String url = null;
		
			
			
			//against which environment are we using the service
			String env = configuration.getEnvironment();

			//must be US
			if(!store.getCountry().getIsoCode().equals("US")) {
				throw new IntegrationException("Can't use the service for store country code ");
			}

			Map<String, ModuleConfig> moduleConfigsMap = module.getModuleConfigs();
			for(String key : moduleConfigsMap.keySet()) {
				
				ModuleConfig moduleConfig = moduleConfigsMap.get(key);
				if(moduleConfig.getEnv().equals(env)) {
					host = moduleConfig.getHost();
					protocol = moduleConfig.getScheme();
					port = moduleConfig.getPort();
					url = moduleConfig.getUri();
				}
			}
			

			StringBuilder xmlheader = new StringBuilder();
			if(store.getCountry().getIsoCode().equals(delivery.getCountry().getIsoCode())) {
				xmlheader.append("<RateV3Request USERID=\"").append(keys.get("account")).append("\">");
			} else {
				xmlheader.append("<IntlRateRequest USERID=\"").append(keys.get("account")).append("\">");
			}



			StringBuilder xmldatabuffer = new StringBuilder();


			double totalW = 0;
			double totalH = 0;
			double totalL = 0;
			double totalG = 0;
			double totalP = 0;

			for (PackageDetails detail : packages) {


				// need size in inch
				double w = DataUtils.getMeasure(detail.getShippingWidth(),
						store, MeasureUnit.IN.name());
				double h = DataUtils.getMeasure(detail.getShippingHeight(),
						store, MeasureUnit.IN.name());
				double l = DataUtils.getMeasure(detail.getShippingLength(),
						store, MeasureUnit.IN.name());
	
				totalW = totalW + w;
				totalH = totalH + h;
				totalL = totalL + l;
	
				// Girth = Length + (Width x 2) + (Height x 2)
				double girth = l + (w * 2) + (h * 2);
		
				totalG = totalG + girth;
	
				// need weight in pounds
				double p = DataUtils.getWeight(detail.getShippingWeight(), store, MeasureUnit.LB.name());
	
				totalP = totalP + p;

			}

/*			BigDecimal convertedOrderTotal = CurrencyUtil.convertToCurrency(
					orderTotal, store.getCurrency(),
					Constants.CURRENCY_CODE_USD);*/

			// calculate total shipping volume

			// ship date is 3 days from here

			Calendar c = Calendar.getInstance();
			c.setTime(new Date());
			c.add(Calendar.DATE, 3);
			Date newDate = c.getTime();
			
			SimpleDateFormat format = new SimpleDateFormat(Constants.DEFAULT_DATE_FORMAT);
			String shipDate = format.format(newDate);
			


			int i = 1;

			// need pounds and ounces
			int pounds = (int) totalP;
			String ouncesString = String.valueOf(totalP - pounds);
			int ouncesIndex = ouncesString.indexOf(".");
			String ounces = "00";
			if (ouncesIndex > -1) {
				ounces = ouncesString.substring(ouncesIndex + 1);
			}

			String size = "REGULAR";
		
			if (totalL + totalG <= 64) {
				size = "REGULAR";
			} else if (totalL + totalG <= 108) {
				size = "LARGE";
			} else {
				size = "OVERSIZE";
			}

			/**
			 * Domestic <Package ID="1ST"> <Service>ALL</Service>
			 * <ZipOrigination>90210</ZipOrigination>
			 * <ZipDestination>96698</ZipDestination> <Pounds>8</Pounds>
			 * <Ounces>32</Ounces> <Container/> <Size>REGULAR</Size>
			 * <Machinable>true</Machinable> </Package>
			 * 
			 * //MAXWEIGHT=70 lbs
			 * 
			 * 
			 * //domestic container default=VARIABLE whiteSpace=collapse
			 * enumeration=VARIABLE enumeration=FLAT RATE BOX enumeration=FLAT
			 * RATE ENVELOPE enumeration=LG FLAT RATE BOX
			 * enumeration=RECTANGULAR enumeration=NONRECTANGULAR
			 * 
			 * //INTL enumeration=Package enumeration=Postcards or aerogrammes
			 * enumeration=Matter for the blind enumeration=Envelope
			 * 
			 * Size May be left blank in situations that do not Size. Defined as
			 * follows: REGULAR: package plus girth is 84 inches or less; LARGE:
			 * package length plus girth measure more than 84 inches not more
			 * than 108 inches; OVERSIZE: package length plus girth is more than
			 * 108 but not 130 inches. For example: <Size>REGULAR</Size>
			 * 
			 * International <Package ID="1ST"> <Machinable>true</Machinable>
			 * <MailType>Envelope</MailType> <Country>Canada</Country>
			 * <Length>0</Length> <Width>0</Width> <Height>0</Height>
			 * <ValueOfContents>250</ValueOfContents> </Package>
			 * 
			 * <Package ID="2ND"> <Pounds>4</Pounds> <Ounces>3</Ounces>
			 * <MailType>Package</MailType> <GXG> <Length>46</Length>
			 * <Width>14</Width> <Height>15</Height> <POBoxFlag>N</POBoxFlag>
			 * <GiftFlag>N</GiftFlag> </GXG>
			 * <ValueOfContents>250</ValueOfContents> <Country>Japan</Country>
			 * </Package>
			 */

			xmldatabuffer.append("<Package ID=\"").append(i).append("\">");


			if(store.getCountry().getIsoCode().equals(delivery.getCountry().getIsoCode())) {

				xmldatabuffer.append("<Service>");
				xmldatabuffer.append("ALL");
				xmldatabuffer.append("</Service>");
				xmldatabuffer.append("<ZipOrigination>");
				xmldatabuffer.append(DataUtils
						.trimPostalCode(store.getStorepostalcode()));
				xmldatabuffer.append("</ZipOrigination>");
				xmldatabuffer.append("<ZipDestination>");
				xmldatabuffer.append(DataUtils
						.trimPostalCode(delivery.getPostalCode()));
				xmldatabuffer.append("</ZipDestination>");
				xmldatabuffer.append("<Pounds>");
				xmldatabuffer.append(pounds);
				xmldatabuffer.append("</Pounds>");
				xmldatabuffer.append("<Ounces>");
				xmldatabuffer.append(ounces);
				xmldatabuffer.append("</Ounces>");
				xmldatabuffer.append("<Container>");
				xmldatabuffer.append(pack);
				xmldatabuffer.append("</Container>");
				xmldatabuffer.append("<Size>");
				xmldatabuffer.append(size);
				xmldatabuffer.append("</Size>");
				xmldatabuffer.append("<Machinable>true</Machinable>");//TODO must be changed if not machinable
				xmldatabuffer.append("<ShipDate>");
				xmldatabuffer.append(shipDate);
				xmldatabuffer.append("</ShipDate>");
			} else {
				// if international
				xmldatabuffer.append("<Pounds>");
				xmldatabuffer.append(pounds);
				xmldatabuffer.append("</Pounds>");
				xmldatabuffer.append("<Ounces>");
				xmldatabuffer.append(ounces);
				xmldatabuffer.append("</Ounces>");
				xmldatabuffer.append("<MailType>");
				xmldatabuffer.append(pack);
				xmldatabuffer.append("</MailType>");
				xmldatabuffer.append("<ValueOfContents>");
				xmldatabuffer.append(productPriceUtils.getAdminFormatedAmount(store, orderTotal));
				xmldatabuffer.append("</ValueOfContents>");
				xmldatabuffer.append("<Country>");
				xmldatabuffer.append(destination.getName());
				xmldatabuffer.append("</Country>");
			}

			// if international & CXG
			/*
			 * xmldatabuffer.append("<CXG>"); xmldatabuffer.append("<Length>");
			 * xmldatabuffer.append(""); xmldatabuffer.append("</Length>");
			 * xmldatabuffer.append("<Width>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</Width>");
			 * xmldatabuffer.append("<Height>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</Height>");
			 * xmldatabuffer.append("<POBoxFlag>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</POBoxFlag>");
			 * xmldatabuffer.append("<GiftFlag>"); xmldatabuffer.append("");
			 * xmldatabuffer.append("</GiftFlag>");
			 * xmldatabuffer.append("</CXG>");
			 */
		
			/*
			 * xmldatabuffer.append("<Width>"); xmldatabuffer.append(totalW);
			 * xmldatabuffer.append("</Width>");
			 * xmldatabuffer.append("<Length>"); xmldatabuffer.append(totalL);
			 * xmldatabuffer.append("</Length>");
			 * xmldatabuffer.append("<Height>"); xmldatabuffer.append(totalH);
			 * xmldatabuffer.append("</Height>");
			 * xmldatabuffer.append("<Girth>"); xmldatabuffer.append(totalG);
			 * xmldatabuffer.append("</Girth>");
			 */

			xmldatabuffer.append("</Package>");

			String xmlfooter = "</RateV3Request>";
			if(!store.getCountry().getIsoCode().equals(delivery.getCountry().getIsoCode())) {
				xmlfooter = "</IntlRateRequest>";
			}

			StringBuilder xmlbuffer = new StringBuilder().append(xmlheader.toString()).append(
					xmldatabuffer.toString()).append(xmlfooter);

			LOGGER.debug("USPS QUOTE REQUEST " + xmlbuffer.toString());
			//HttpClient client = new HttpClient();
			try(CloseableHttpClient httpclient = HttpClients.createDefault()) {
			@SuppressWarnings("deprecation")
			String encoded = java.net.URLEncoder.encode(xmlbuffer.toString());

			String completeUri = url + "?API=RateV3&XML=" + encoded;
			if(!store.getCountry().getIsoCode().equals(delivery.getCountry().getIsoCode())) {
				completeUri = url + "?API=IntlRate&XML=" + encoded;
			}

			// ?API=RateV3

			httpget = new HttpGet(protocol + "://" + host + ":" + port
					+ completeUri);
			// RequestEntity entity = new
			// StringRequestEntity(xmlbuffer.toString(),"text/plain","UTF-8");
			// httpget.setRequestEntity(entity);

            ResponseHandler<String> responseHandler = response -> {
				int status = response.getStatusLine().getStatusCode();
				if (status >= 200 && status < 300) {
					HttpEntity entity = response.getEntity();
					return entity != null ? EntityUtils.toString(entity) : null;
				} else {
					LOGGER.error("Communication Error with ups quote " + status);
					throw new ClientProtocolException("UPS quote communication error " + status);
				}
			};

            String data = httpclient.execute(httpget, responseHandler);
/*			int result = client.executeMethod(httpget);
			if (result != 200) {
				LOGGER.error("Communication Error with usps quote " + result + " "
						+ protocol + "://" + host + ":" + port + url);
				throw new Exception("USPS quote communication error " + result);
			}*/
			//data = httpget.getResponseBodyAsString();
			LOGGER.debug("usps quote response " + data);

			USPSParsedElements parsed = new USPSParsedElements();

			/**
			 * <RateV3Response> <Package ID="1ST">
			 * <ZipOrigination>44106</ZipOrigination>
			 * <ZipDestination>20770</ZipDestination>
			 */

			Digester digester = new Digester();
			digester.push(parsed);

			if(store.getCountry().getIsoCode().equals(delivery.getCountry().getIsoCode())) {

				digester.addCallMethod("Error/Description",
						"setError", 0);
				digester.addCallMethod("RateV3Response/Package/Error/Description",
						"setError", 0);
				digester
						.addObjectCreate(
								"RateV3Response/Package/Postage",
								ShippingOption.class);
				digester.addSetProperties("RateV3Response/Package/Postage",
						"CLASSID", "optionId");
				digester.addCallMethod(
						"RateV3Response/Package/Postage/MailService",
						"setOptionName", 0);
				digester.addCallMethod(
						"RateV3Response/Package/Postage/MailService",
						"setOptionCode", 0);
				digester.addCallMethod("RateV3Response/Package/Postage/Rate",
						"setOptionPriceText", 0);
				//digester
				//		.addCallMethod(
				//				"RateV3Response/Package/Postage/Commitment/CommitmentDate",
				//				"estimatedNumberOfDays", 0);
				digester.addSetNext("RateV3Response/Package/Postage",
						"addOption");

			} else {

				digester.addCallMethod("Error/Description",
						"setError", 0);
				digester.addCallMethod("IntlRateResponse/Package/Error/Description",
						"setError", 0);
				digester
						.addObjectCreate(
								"IntlRateResponse/Package/Service",
								ShippingOption.class);
				digester.addSetProperties("IntlRateResponse/Package/Service",
						"ID", "optionId");
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/SvcDescription",
						"setOptionName", 0);
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/SvcDescription",
						"setOptionCode", 0);
				digester.addCallMethod(
						"IntlRateResponse/Package/Service/Postage",
						"setOptionPriceText", 0);
				//digester.addCallMethod(
				//		"IntlRateResponse/Package/Service/SvcCommitments",
				//		"setEstimatedNumberOfDays", 0);
				digester.addSetNext("IntlRateResponse/Package/Service",
						"addOption");

			}

			// <?xml
			// version="1.0"?><AddressValidationResponse><Response><TransactionReference><CustomerContext>SalesManager
			// Data</CustomerContext><XpciVersion>1.0</XpciVersion></TransactionReference><ResponseStatusCode>0</ResponseStatusCode><ResponseStatusDescription>Failure</ResponseStatusDescription><Error><ErrorSeverity>Hard</ErrorSeverity><ErrorCode>10002</ErrorCode><ErrorDescription>The
			// XML document is well formed but the document is not
			// valid</ErrorDescription><ErrorLocation><ErrorLocationElementName>AddressValidationRequest</ErrorLocationElementName></ErrorLocation></Error></Response></AddressValidationResponse>


			//<?xml version="1.0"?>
			//<IntlRateResponse><Package ID="1"><Error><Number>-2147218046</Number>
			//<Source>IntlPostage;clsIntlPostage.GetCountryAndRestirctedServiceId;clsIntlPostage.CalcAllPostageDimensionsXML;IntlRate.ProcessRequest</Source>
			//<Description>Invalid Country Name</Description><HelpFile></HelpFile><HelpContext>1000440</HelpContext></Error></Package></IntlRateResponse>


			xmlreader = new StringReader(data);
			digester.parse(xmlreader);

			if (!StringUtils.isBlank(parsed.getError())) {
				LOGGER.error("Can't process USPS message= "
						+ parsed.getError());
				throw new IntegrationException(parsed.getError());
			}
			if (!StringUtils.isBlank(parsed.getStatusCode())
					&& !parsed.getStatusCode().equals("1")) {
				LOGGER.error("Can't process USPS statusCode="
						+ parsed.getStatusCode() + " message= "
						+ parsed.getError());
				throw new IntegrationException(parsed.getError());
			}

			if (parsed.getOptions() == null || parsed.getOptions().size() == 0) {
				LOGGER.warn("No options returned from USPS");
				throw new IntegrationException(parsed.getError());
			}



/*			String carrier = getShippingMethodDescription(locale);
			// cost is in USD, need to do conversion

			MerchantConfiguration rtdetails = config
					.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_DISPLAY_REALTIME_QUOTES);
			int displayQuoteDeliveryTime = ShippingConstants.NO_DISPLAY_RT_QUOTE_TIME;
			if (rtdetails != null) {

				if (!StringUtils.isBlank(rtdetails.getConfigurationValue1())) {// display
																				// or
																				// not
																				// quotes
					try {
						displayQuoteDeliveryTime = Integer.parseInt(rtdetails
								.getConfigurationValue1());

					} catch (Exception e) {
						log.error("Display quote is not an integer value ["
								+ rtdetails.getConfigurationValue1() + "]");
					}
				}
			}

			LabelUtil labelUtil = LabelUtil.getInstance();*/
			// Map serviceMap =
			// com.salesmanager.core.util.ShippingUtil.buildServiceMap("usps",locale);

			@SuppressWarnings("unchecked")
			List<ShippingOption> shippingOptions = parsed.getOptions();

/*			List<ShippingOption> returnOptions = null;

			if (shippingOptions != null && shippingOptions.size() > 0) {

				returnOptions = new ArrayList<ShippingOption>();
				// Map selectedintlservices =
				// (Map)config.getConfiguration("service-global-usps");
				// need to create a Map of LABEL - LABLEL
				// Iterator servicesIterator =
				// selectedintlservices.keySet().iterator();
				// Map services = new HashMap();

				// ResourceBundle bundle = ResourceBundle.getBundle("usps",
				// locale);

				// while(servicesIterator.hasNext()) {
				// String key = (String)servicesIterator.next();
				// String value =
				// bundle.getString("shipping.quote.services.label." + key);
				// services.put(value, key);
				// }

				for(ShippingOption option : shippingOptions) {

					StringBuilder description = new StringBuilder();
					description.append(option.getOptionName());
					//if (displayQuoteDeliveryTime == ShippingConstants.DISPLAY_RT_QUOTE_TIME) {
					if (shippingConfiguration.getShippingDescription()==ShippingDescription.LONG_DESCRIPTION) {
						if (option.getEstimatedNumberOfDays()>0) {
							description.append(" (").append(
									option.getEstimatedNumberOfDays()).append(
									" ").append(
									" d")
									.append(")");
						}
					}
					option.setDescription(description.toString());

					// get currency
					if (!option.getCurrency().equals(store.getCurrency())) {
						option.setOptionPrice(CurrencyUtil.convertToCurrency(
								option.getOptionPrice(), option.getCurrency(),
								store.getCurrency()));
					}

					// if(!services.containsKey(option.getOptionCode())) {
					// if(returnColl==null) {
					// returnColl = new ArrayList();
					// }
					// returnColl.add(option);
					// }
					returnOptions.add(option);
				}

				// if(options.size()==0) {
				// CommonService.logServiceMessage(store.getMerchantId(),
				// " none of the service code returned by UPS [" +
				// selectedintlservices.keySet().toArray(new
				// String[selectedintlservices.size()]) +
				// "] for this shipping is in your selection list");
				// }

			}*/

			return shippingOptions;
			}

		} catch (Exception e1) {
			LOGGER.error("Error in USPS shipping quote ",e1);
			throw new IntegrationException(e1);
		} finally {
			if (xmlreader != null) {
				try {
					xmlreader.close();
				} catch (Exception ignore) {
				}
			}
			if (httpget != null) {
				httpget.releaseConnection();
			}
		}


	}



	@Override
	public CustomIntegrationConfiguration getCustomModuleConfiguration(
			MerchantStore store) throws IntegrationException {
		//nothing to do
		return null;
	}

}


class USPSParsedElements {

	private String statusCode;
	private String statusMessage;
	private String error = "";
	private String errorCode = "";
	private List<ShippingOption> options = new ArrayList<ShippingOption>();

	public void addOption(ShippingOption option) {
		options.add(option);
	}

	public List getOptions() {
		return options;
	}

	public String getStatusCode() {
		return statusCode;
	}

	public void setStatusCode(String statusCode) {
		this.statusCode = statusCode;
	}

	public String getStatusMessage() {
		return statusMessage;
	}

	public void setStatusMessage(String statusMessage) {
		this.statusMessage = statusMessage;
	}

	public String getError() {
		return error;
	}

	public void setError(String error) {
		this.error = error;
	}

	public String getErrorCode() {
		return errorCode;
	}

	public void setErrorCode(String errorCode) {
		this.errorCode = errorCode;
	}

}



```
