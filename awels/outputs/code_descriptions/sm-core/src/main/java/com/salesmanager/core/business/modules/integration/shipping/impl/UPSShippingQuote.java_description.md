# UPSShippingQuote.java

## Review

## 1. Summary  
**Purpose** – The `UPSShippingQuote` class implements the `ShippingQuoteModule` interface and is responsible for communicating with the UPS Rating API to obtain shipping rates for a given order.  
**Key Components**  
| Component | Role |
|-----------|------|
| `validateModuleConfiguration` | Checks that the configuration supplied to the module contains the required UPS credentials and at least one package type. |
| `getShippingQuotes` | Builds an XML request, sends it to UPS, parses the XML response and returns a list of `ShippingOption` objects that the front‑end can display. |
| `getCustomModuleConfiguration` | Placeholder – currently returns `null`. |
| `UPSParsedElements` | Simple POJO used by the Apache Commons Digester to capture parsed data from the UPS response. |

**Design patterns / libraries**  
* **Builder pattern (indirect)** – XML request is constructed by repeatedly appending to a `StringBuilder`.  
* **Template method** – The class implements a well‑defined interface (`ShippingQuoteModule`).  
* **Apache Commons Digester** – Used for parsing the XML response into a POJO.  
* **Apache HttpClient** – For HTTP communication.  
* **SLF4J** – Logging.  
* **Bean Validation** – `org.apache.commons.lang3.Validate` is used to guard against `null` configuration objects.  

---

## 2. Detailed Description  

### Flow of execution  

1. **Configuration validation**  
   * `validateModuleConfiguration` is invoked before any shipping quotes are requested.  
   * It verifies that the keys `accessKey`, `userId`, and `password` are present and non‑blank.  
   * It also verifies that the `packages` integration option exists and is non‑empty.  
   * If any of these checks fail, an `IntegrationException` is thrown containing the missing field names.  

2. **Quote request**  
   * `getShippingQuotes` is called with order, package, delivery, origin, store, and configuration objects.  
   * Basic checks: postal code must exist, packages must be provided, store country must be US or CA, language limited to EN/FR.  
   * The module looks up the UPS endpoint (host, scheme, port, uri) for the current environment (e.g., “production” or “sandbox”).  
   * A request XML is built manually:
     * Access request block (license, user, password).  
     * `RatingServiceSelectionRequest` with transaction reference and action.  
     * `Shipment` details (shipper, ship‑to, package weight, dimensions, packaging type).  
   * The XML is posted to the UPS endpoint using `CloseableHttpClient`.  
   * The response is parsed with a `Digester` that populates a `UPSParsedElements` instance.  
   * Errors returned by UPS (status code != “1” or explicit error elements) result in an `IntegrationException`.  
   * Parsed `ShippingOption` objects are enriched with the human‑readable name from the module’s details map and converted to `BigDecimal` prices.  
   * The resulting list of `ShippingOption` objects is returned.  

3. **Cleanup**  
   * The `HttpPost` connection is released in a `finally` block.  
   * The `reader` (never used) is closed if it was ever instantiated.  

### Assumptions & Constraints  

| Item | Assumption | Impact |
|------|------------|--------|
| Country codes | Only US or CA are supported | Calls to UPS for other countries will immediately return `null` |
| `store.getZone()` / `delivery.getZone()` | Non‑null when available | NullPointerException if zones are missing |
| `packageDetail` fields | Non‑null strings | NullPointerException on `new BigDecimal(...)` |
| Locale | Only EN or FR | Other locales default to EN |
| Currency | Prices returned in CAD (assumed) | No conversion logic – currency mismatch can produce wrong display |
| HTTP endpoint | Provided in module config | Missing config leads to `NullPointerException` when building URL |
| Digester | XML structure matches expectations | Any change in UPS API will break parsing |

### Architecture & Design Choices  

* The module is tightly coupled to the UPS Rating API.  
* XML request and response handling are done manually rather than with a dedicated XML library (e.g., JAXB).  
* The code uses a `try`‑with‑resources block for the `HttpClient` but still manually releases the `HttpPost`.  
* Error handling is rudimentary – all non‑200 HTTP statuses or any UPS error message results in a generic `IntegrationException`.  
* The class relies on a `Map<String, ModuleConfig>` supplied by the `IntegrationModule` to locate the appropriate endpoint; this is a lightweight form of dependency injection.  

---

## 3. Functions/Methods  

| Method | Signature | Responsibility | Notable Observations |
|--------|-----------|----------------|----------------------|
| **`validateModuleConfiguration`** | `void validateModuleConfiguration(IntegrationConfiguration configuration, MerchantStore store)` | Validates the configuration supplied to UPS. | Re‑initialises `errorFields` on every missing key, therefore only the last missing field is reported. |
| **`getShippingQuotes`** | `List<ShippingOption> getShippingQuotes(Order order, List<ShippingPackage> packages, ShippingDelivery delivery, ShippingOrigin origin, MerchantStore store, IntegrationConfiguration integrationConfiguration, ShippingMethod shippingMethod, Locale locale)` | Core method that contacts UPS, parses the response and returns a list of `ShippingOption` objects. | Handles most of the module’s logic: XML building, HTTP call, response parsing, enrichment of options. |
| **`getCustomModuleConfiguration`** | `CustomIntegrationConfiguration getCustomModuleConfiguration(MerchantStore store)` | Placeholder – returns `null`. | Indicates that the UPS module currently does not expose a custom UI for configuration. |
| **`UPSParsedElements.addOption(ShippingOption)`** | `void addOption(ShippingOption option)` | Adds an option to the internal list during Digester parsing. | Simple helper – no side effects. |
| **`UPSParsedElements.getOptions`** | `List<ShippingOption> getOptions()` | Returns the list of parsed options. | Used after Digester completes. |

### Inner class `UPSParsedElements`  

* Plain data holder with getters/setters for status, error, and a list of `ShippingOption`.  
* `addOption` is used by the Digester (`addSetNext`) to aggregate options.  

---

## 4. Dependencies  

| Library | Purpose |
|---------|---------|
| `org.apache.commons.lang3.StringUtils` | String utilities (`isBlank`, `isNotBlank`) |
| `org.apache.commons.lang3.Validate` | Simple non‑null guard |
| `org.apache.http.client.methods.HttpPost` | HTTP request object |
| `org.apache.http.impl.client.CloseableHttpClient` & `HttpClients` | Apache HttpClient for REST calls |
| `org.apache.commons.digester3.Digester` | XML parsing into POJO |
| `org.slf4j.Logger` & `LoggerFactory` | Logging (SLF4J) |
| `java.util.*` | Collections (`List`, `Map`, `ArrayList`, `HashMap`) |
| `java.io.*` | `StringReader`, `Reader` for Digester |
| `java.math.BigDecimal` | Monetary amounts |
| `com.salesmanager.core.model.*` | Domain entities (`MerchantStore`, `ShippingOption`, etc.) |
| `com.salesmanager.core.business.*` | Integration framework (`IntegrationModule`, `IntegrationConfiguration`, `IntegrationException`) |

---

## 4. Additional Notes – Issues & Recommendations  

### 4.1 Configuration Validation  
* **Problem** – `errorFields` is re‑created on each failure, meaning only the last missing field is reported.  
* **Fix** – Instantiate once and add to it for every missing key or option.  
* **Other** – Validation for `services` is commented out but still referenced; either re‑enable it or remove the dead‑code.  

### 4.2 XML Request Construction  
* **Hard‑coded concatenation** can lead to malformed XML if any value contains characters that need escaping (e.g., `<`, `&`).  
* **Recommendation** – Use an XML library (JAXB, XStream, or even a simple `org.w3c.dom.Document` with a `Transformer`) to build the request, which automatically handles escaping.  
* **Maintainability** – The current approach is brittle; any change in the UPS API (e.g., new element names) will break the string concatenation logic.  

### 4.3 Null‑Pointer Risks  
* `store.getZone()` and `delivery.getZone()` are dereferenced without null checks.  
* `packageDetail.getShippingWeight()`, `getShippingLength()`, `getShippingWidth()`, `getShippingHeight()` are used directly in `new BigDecimal(...)`.  
* If `moduleConfig` for the current environment is missing, `host`, `protocol`, `port`, or `url` will be `null` and a `NullPointerException` will occur when building the request URL.  
* **Fix** – Add defensive checks and throw a clear `IntegrationException` if required data is missing.  

### 4.4 Error Handling & Logging  
* All exceptions (including `IOException`, `XMLException`, `NumberFormatException`) are caught as a generic `Exception` and wrapped into `IntegrationException`.  
* Logging is only at `debug` level for the request and `error` level for failures.  
* **Recommendation** – Provide more granular exception types (e.g., `HttpCommunicationException`, `UpsApiException`, `ConfigurationException`).  
* Add stack‑trace logging for HTTP errors and XML parsing failures.  

### 4.5 Currency Handling  
* The code assumes that prices returned by UPS are in CAD, yet the front‑end may display in the store’s currency.  
* No conversion logic is present.  
* **Recommendation** – Either expose a configuration flag that indicates whether the store currency is CAD or implement a currency conversion routine that consults an external exchange‑rate service.  

### 4.6 Unused Variables & Dead Code  
* `Reader reader` is declared but never used; the associated `finally` block that closes it is unnecessary.  
* Large blocks of commented‑out legacy code (service mapping, real‑time quote display logic, etc.) clutter the file.  
* **Recommendation** – Remove unused variables and dead code to improve readability and reduce maintenance burden.  

### 4.7 API Extensibility  
* The class is an implementation of a single external API (UPS). If the organization decides to support another carrier, a new module will need to be written from scratch.  
* **Recommendation** – Abstract the XML construction and parsing logic into reusable helper classes (e.g., `XmlRequestBuilder`, `XmlResponseParser`) so that each carrier implementation can reuse common functionality.  

### 4.8 Code Style  
* In `validateModuleConfiguration`, the repeated `new ArrayList<>()` inside each `if` block hides earlier errors.  
* The use of `BigDecimal.ROUND_HALF_UP` is fine but modern Java recommends `RoundingMode.HALF_UP`.  
* The class is a single Java file with an inner helper class; consider moving `UPSParsedElements` into its own file for clarity.  

---

## 4. Recommendations  

| Issue | Suggested Fix | Rationale |
|-------|---------------|-----------|
| **Missing/overwritten error fields** | Aggregate all missing fields into a single `List` | Gives the caller a full list of problems instead of one at a time |
| **Potential NPE on zones & package fields** | Guard against `null` before dereferencing (`zone != null ? zone.getCode() : null`) | Avoid crashes in real‑world scenarios where zone information is incomplete |
| **Manual XML handling** | Switch to JAXB, XStream or a DOM builder | Guarantees well‑formed XML, easier maintenance, and automatic escaping |
| **Digester error handling** | Catch `IOException` & `SAXException` from `digester.parse()` and wrap in `IntegrationException` | Makes failures explicit and traceable |
| **HTTP status handling** | Use `ResponseHandler` that throws `HttpResponseException` for non‑200 statuses; log the status and response body | Provides richer diagnostic information |
| **Unused `reader` & `reader.close()`** | Remove the `reader` variable entirely | Simplifies the `finally` block |
| **Null‑safe `HttpPost` release** | Verify `httppost != null` before calling `releaseConnection()` | Avoids NPE in the `finally` block |
| **Currency conversion stub** | Either remove the comment or implement a simple conversion if the store currency differs from CAD | Prevents hidden price errors |
| **Locale‑agnostic number parsing** | Use `NumberFormat.getInstance(locale)` when converting `priceText` to `BigDecimal` | Handles locales where the decimal separator is `,` |
| **Logging** | Add `trace` or `debug` logs for request/response payloads in a secure way (mask credentials) | Improves troubleshooting while keeping credentials protected |
| **Method signatures** | Return an empty list instead of `null` when no quotes are found | Simplifies consumer logic (no need to check for `null`) |

---

## 4. Dependencies (continued)  

| External Library | Typical Version | Role in the module |
|------------------|-----------------|--------------------|
| **Apache Commons Lang** | 3.x | `Validate`, `StringUtils` |
| **Apache Commons Digester** | 3.x | XML parsing |
| **Apache HttpClient** | 4.5.x | HTTP communication (with `try‑with‑resources` for `CloseableHttpClient`) |
| **SLF4J** | 1.7.x | Logging abstraction |
| **SalesManager Core** | – | Domain models (`MerchantStore`, `ShippingOption`, etc.) and integration framework (`IntegrationModule`, `IntegrationConfiguration`) |

---

## 5. Additional Notes  

### Performance  
* The XML request is built in memory via `StringBuilder`; for small requests this is fine, but for large orders the concatenation can become expensive.  
* Digester is lightweight but parses the entire XML string twice (`StringReader` → `digester.parse`).  

### Security  
* The request XML contains sensitive credentials (`accessKey`, `userId`, `password`).  
* Logging prints the full XML to debug level. If logs are not rotated or masked, credentials could be exposed.  
* **Recommendation** – Mask or strip the credential fields before logging the XML payload.  

### Unit & Integration Testing  
* The current implementation is hard to unit‑test because XML creation/parsing is done manually.  
* Suggestion: Introduce an abstraction (`UpsXmlBuilder`, `UpsXmlParser`) and inject test doubles.  
* Use of `IntegrationException` as a generic error type makes it hard to assert on specific failure modes.  

### Readability & Maintenance  
* The file contains large commented blocks that reference legacy logic (currency conversion, real‑time quote display).  
* Consider extracting the XML request generation into a dedicated builder class, and the response parsing into a separate parser class.  
* Move `UPSParsedElements` to its own file to keep the public API focused.  

### Compliance with Java 17+  
* Use of `ClientProtocolException` is deprecated; replace with `IOException`.  
* Use `RoundingMode.HALF_UP` instead of the legacy `BigDecimal.ROUND_HALF_UP`.  

---

### Bottom line  
`UPSShippingQuote` is functional and follows the overall architecture of the SalesManager integration framework. However, it is fragile: a handful of `null` dereferences, inadequate configuration validation, manual XML handling, and blunt error reporting make it prone to runtime failures when the surrounding data is incomplete or when UPS changes its API. Addressing the above recommendations will make the module more robust, maintainable, and easier to test.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.io.BufferedReader;
import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.commons.digester.Digester;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.apache.http.HttpEntity;
import org.apache.http.client.ClientProtocolException;
import org.apache.http.client.ResponseHandler;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.ContentType;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.utils.DataUtils;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
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
 * Integrates with UPS online API
 * @author casams1
 *
 */
public class UPSShippingQuote implements ShippingQuoteModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(UPSShippingQuote.class);


	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		
		
		List<String> errorFields = null;
		
		//validate integrationKeys['accessKey']
		Map<String,String> keys = integrationConfiguration.getIntegrationKeys();
		if(keys==null || StringUtils.isBlank(keys.get("accessKey"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("accessKey");
		}
		
		if(keys==null || StringUtils.isBlank(keys.get("userId"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("userId");
		}
		
		if(keys==null || StringUtils.isBlank(keys.get("password"))) {
			errorFields = new ArrayList<String>();
			errorFields.add("password");
		}

		//validate at least one integrationOptions['packages']
		Map<String,List<String>> options = integrationConfiguration.getIntegrationOptions();
		if(options==null) {
			errorFields = new ArrayList<String>();
			errorFields.add("packages");
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
		
		Validate.notNull(configuration, "IntegrationConfiguration must not be null for USPS shipping module");

		
		if(StringUtils.isBlank(delivery.getPostalCode())) {
			return null;
		}

		BigDecimal total = orderTotal;

		if (packages == null) {
			return null;
		}
		
		List<ShippingOption> options = null;

		// only applies to Canada and US
		Country country = delivery.getCountry();
		

		
		if(!(country.getIsoCode().equals("US") || country.getIsoCode().equals("CA"))) {
			return null;
			//throw new IntegrationException("UPS Not configured for shipping in country " + country.getIsoCode());
		}

		// supports en and fr
		String language = locale.getLanguage();
		if (!language.equals(Locale.FRENCH.getLanguage())
				&& !language.equals(Locale.ENGLISH.getLanguage())) {
			language = Locale.ENGLISH.getLanguage();
		}
		
		String pack = configuration.getIntegrationOptions().get("packages").get(0);
		Map<String,String> keys = configuration.getIntegrationKeys();
		
		String accessKey = keys.get("accessKey");
		String userId = keys.get("userId");
		String password = keys.get("password");
		
		
		String host = null;
		String protocol = null;
		String port = null;
		String url = null;
		
		StringBuilder xmlbuffer = new StringBuilder();
		HttpPost httppost = null;
		BufferedReader reader = null;

		try {
			String env = configuration.getEnvironment();
			
			Set<String> regions = module.getRegionsSet();
			if(!regions.contains(store.getCountry().getIsoCode())) {
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

			
			StringBuilder xmlreqbuffer = new StringBuilder();
			xmlreqbuffer.append("<?xml version=\"1.0\"?>");
			xmlreqbuffer.append("<AccessRequest>");
			xmlreqbuffer.append("<AccessLicenseNumber>");
			xmlreqbuffer.append(accessKey);
			xmlreqbuffer.append("</AccessLicenseNumber>");
			xmlreqbuffer.append("<UserId>");
			xmlreqbuffer.append(userId);
			xmlreqbuffer.append("</UserId>");
			xmlreqbuffer.append("<Password>");
			xmlreqbuffer.append(password);
			xmlreqbuffer.append("</Password>");
			xmlreqbuffer.append("</AccessRequest>");
			
			String xmlhead = xmlreqbuffer.toString();
			

			String weightCode = store.getWeightunitcode();
			String measureCode = store.getSeizeunitcode();

			if (weightCode.equals("KG")) {
				weightCode = "KGS";
			} else {
				weightCode = "LBS";
			}

			String xml = "<?xml version=\"1.0\"?><RatingServiceSelectionRequest><Request><TransactionReference><CustomerContext>Shopizer</CustomerContext><XpciVersion>1.0001</XpciVersion></TransactionReference><RequestAction>Rate</RequestAction><RequestOption>Shop</RequestOption></Request>";
			StringBuilder xmldatabuffer = new StringBuilder();

			/**
			 * <Shipment>
			 * 
			 * <Shipper> <Address> <City></City>
			 * <StateProvinceCode>QC</StateProvinceCode>
			 * <CountryCode>CA</CountryCode> <PostalCode></PostalCode>
			 * </Address> </Shipper>
			 * 
			 * <ShipTo> <Address> <City>Redwood Shores</City>
			 * <StateProvinceCode>CA</StateProvinceCode>
			 * <CountryCode>US</CountryCode> <PostalCode></PostalCode>
			 * <ResidentialAddressIndicator/> </Address> </ShipTo>
			 * 
			 * <Package> <PackagingType> <Code>21</Code> </PackagingType>
			 * <PackageWeight> <UnitOfMeasurement> <Code>LBS</Code>
			 * </UnitOfMeasurement> <Weight>1.1</Weight> </PackageWeight>
			 * <PackageServiceOptions> <InsuredValue>
			 * <CurrencyCode>CAD</CurrencyCode>
			 * <MonetaryValue>100</MonetaryValue> </InsuredValue>
			 * </PackageServiceOptions> </Package>
			 * 
			 * 
			 * </Shipment>
			 * 
			 * <CustomerClassification> <Code>03</Code>
			 * </CustomerClassification> </RatingServiceSelectionRequest>
			 * **/

			/**Map countriesMap = (Map) RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Map zonesMap = (Map) RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));

			Country storeCountry = (Country) countriesMap.get(store
					.getCountry());

			Country customerCountry = (Country) countriesMap.get(customer
					.getCustomerCountryId());

			int sZone = -1;
			try {
				sZone = Integer.parseInt(store.getZone());
			} catch (Exception e) {
				// TODO: handle exception
			}

			Zone storeZone = (Zone) zonesMap.get(sZone);
			Zone customerZone = (Zone) zonesMap.get(customer
					.getCustomerZoneId());**/
					
				

			xmldatabuffer.append("<PickupType><Code>03</Code></PickupType>");
			// xmldatabuffer.append("<Description>Daily Pickup</Description>");
			xmldatabuffer.append("<Shipment><Shipper>");
			xmldatabuffer.append("<Address>");
			xmldatabuffer.append("<City>");
			xmldatabuffer.append(store.getStorecity());
			xmldatabuffer.append("</City>");
			// if(!StringUtils.isBlank(store.getStorestateprovince())) {
			if (store.getZone() != null) {
				xmldatabuffer.append("<StateProvinceCode>");
				xmldatabuffer.append(store.getZone().getCode());// zone code
				xmldatabuffer.append("</StateProvinceCode>");
			}
			xmldatabuffer.append("<CountryCode>");
			xmldatabuffer.append(store.getCountry().getIsoCode());
			xmldatabuffer.append("</CountryCode>");
			xmldatabuffer.append("<PostalCode>");
			xmldatabuffer.append(DataUtils
					.trimPostalCode(store.getStorepostalcode()));
			xmldatabuffer.append("</PostalCode></Address></Shipper>");

			// ship to
			xmldatabuffer.append("<ShipTo>");
			xmldatabuffer.append("<Address>");
			xmldatabuffer.append("<City>");
			xmldatabuffer.append(delivery.getCity());
			xmldatabuffer.append("</City>");
			// if(!StringUtils.isBlank(customer.getCustomerState())) {
			if (delivery.getZone() != null) {
				xmldatabuffer.append("<StateProvinceCode>");
				xmldatabuffer.append(delivery.getZone().getCode());// zone code
				xmldatabuffer.append("</StateProvinceCode>");
			}
			xmldatabuffer.append("<CountryCode>");
			xmldatabuffer.append(delivery.getCountry().getIsoCode());
			xmldatabuffer.append("</CountryCode>");
			xmldatabuffer.append("<PostalCode>");
			xmldatabuffer.append(DataUtils
					.trimPostalCode(delivery.getPostalCode()));
			xmldatabuffer.append("</PostalCode></Address></ShipTo>");
			// xmldatabuffer.append("<Service><Code>11</Code></Service>");//TODO service codes (next day ...)


			for(PackageDetails packageDetail : packages){

				xmldatabuffer.append("<Package>");
				xmldatabuffer.append("<PackagingType>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(pack);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</PackagingType>");

				// weight
				xmldatabuffer.append("<PackageWeight>");
				xmldatabuffer.append("<UnitOfMeasurement>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(weightCode);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</UnitOfMeasurement>");
				xmldatabuffer.append("<Weight>");
				xmldatabuffer.append(new BigDecimal(packageDetail.getShippingWeight())
						.setScale(1, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Weight>");
				xmldatabuffer.append("</PackageWeight>");

				// dimension
				xmldatabuffer.append("<Dimensions>");
				xmldatabuffer.append("<UnitOfMeasurement>");
				xmldatabuffer.append("<Code>");
				xmldatabuffer.append(measureCode);
				xmldatabuffer.append("</Code>");
				xmldatabuffer.append("</UnitOfMeasurement>");
				xmldatabuffer.append("<Length>");
				xmldatabuffer.append(new BigDecimal(packageDetail.getShippingLength())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Length>");
				xmldatabuffer.append("<Width>");
				xmldatabuffer.append(new BigDecimal(packageDetail.getShippingWidth())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Width>");
				xmldatabuffer.append("<Height>");
				xmldatabuffer.append(new BigDecimal(packageDetail.getShippingHeight())
						.setScale(2, BigDecimal.ROUND_HALF_UP));
				xmldatabuffer.append("</Height>");
				xmldatabuffer.append("</Dimensions>");
				xmldatabuffer.append("</Package>");

			}

			xmldatabuffer.append("</Shipment>");
			xmldatabuffer.append("</RatingServiceSelectionRequest>");

			xmlbuffer.append(xmlhead).append(xml).append(
					xmldatabuffer.toString());
			


			LOGGER.debug("UPS QUOTE REQUEST " + xmlbuffer.toString());


			try(CloseableHttpClient httpclient = HttpClients.createDefault()) {
			//HttpClient client = new HttpClient();
			httppost = new HttpPost(protocol + "://" + host + ":" + port
					+ url);
			StringEntity entity = new StringEntity(xmlbuffer.toString(),ContentType.APPLICATION_ATOM_XML);
			//RequestEntity entity = new StringRequestEntity(
			//		xmlbuffer.toString(), "text/plain", "UTF-8");
			httppost.setEntity(entity);
            // Create a custom response handler
            ResponseHandler<String> responseHandler = response -> {
				int status = response.getStatusLine().getStatusCode();
				if (status >= 200 && status < 300) {
					HttpEntity entity1 = response.getEntity();
					return entity1 != null ? EntityUtils.toString(entity1) : null;
				} else {
					LOGGER.error("Communication Error with ups quote " + status);
					throw new ClientProtocolException("UPS quote communication error " + status);
				}
			};
			String data = httpclient.execute(httppost, responseHandler);

			//int result = response.getStatusLine().getStatusCode();
			//int result = client.executeMethod(httppost);
/*			if (result != 200) {
				LOGGER.error("Communication Error with ups quote " + result + " "
						+ protocol + "://" + host + ":" + port + url);
				throw new Exception("UPS quote communication error " + result);
			}*/

			LOGGER.debug("ups quote response " + data);

			UPSParsedElements parsed = new UPSParsedElements();

			Digester digester = new Digester();
			digester.push(parsed);
			digester.addCallMethod(
					"RatingServiceSelectionResponse/Response/Error",
					"setErrorCode", 0);
			digester.addCallMethod(
					"RatingServiceSelectionResponse/Response/ErrorDescriprion",
					"setError", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/ResponseStatusCode",
							"setStatusCode", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/ResponseStatusDescription",
							"setStatusMessage", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/Response/Error/ErrorDescription",
							"setError", 0);

			digester.addObjectCreate(
					"RatingServiceSelectionResponse/RatedShipment",
					ShippingOption.class);
			// digester.addSetProperties(
			// "RatingServiceSelectionResponse/RatedShipment", "sequence",
			// "optionId" );
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/Service/Code",
							"setOptionId", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/TotalCharges/MonetaryValue",
							"setOptionPriceText", 0);
			//digester
			//		.addCallMethod(
			//				"RatingServiceSelectionResponse/RatedShipment/TotalCharges/CurrencyCode",
			//				"setCurrency", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/Service/Code",
							"setOptionCode", 0);
			digester
					.addCallMethod(
							"RatingServiceSelectionResponse/RatedShipment/GuaranteedDaysToDelivery",
							"setEstimatedNumberOfDays", 0);
			digester.addSetNext("RatingServiceSelectionResponse/RatedShipment",
					"addOption");

			// <?xml
			// version="1.0"?><AddressValidationResponse><Response><TransactionReference><CustomerContext>SalesManager
			// Data</CustomerContext><XpciVersion>1.0</XpciVersion></TransactionReference><ResponseStatusCode>0</ResponseStatusCode><ResponseStatusDescription>Failure</ResponseStatusDescription><Error><ErrorSeverity>Hard</ErrorSeverity><ErrorCode>10002</ErrorCode><ErrorDescription>The
			// XML document is well formed but the document is not
			// valid</ErrorDescription><ErrorLocation><ErrorLocationElementName>AddressValidationRequest</ErrorLocationElementName></ErrorLocation></Error></Response></AddressValidationResponse>

			Reader xmlreader = new StringReader(data);

			digester.parse(xmlreader);

			if (!StringUtils.isBlank(parsed.getErrorCode())) {

					LOGGER.error("Can't process UPS statusCode="
							+ parsed.getErrorCode() + " message= "
							+ parsed.getError());
				throw new IntegrationException(parsed.getError());
			}
			if (!StringUtils.isBlank(parsed.getStatusCode())
					&& !parsed.getStatusCode().equals("1")) {

				throw new IntegrationException(parsed.getError());
			}

			if (parsed.getOptions() == null || parsed.getOptions().size() == 0) {

				throw new IntegrationException("No shipping options available for the configuration");
			}

			/*String carrier = getShippingMethodDescription(locale);
			// cost is in CAD, need to do conversion

			boolean requiresCurrencyConversion = false; String storeCurrency
			 = store.getCurrency();
			if(!storeCurrency.equals(Constants.CURRENCY_CODE_CAD)) {
			 requiresCurrencyConversion = true; }

			LabelUtil labelUtil = LabelUtil.getInstance();
			Map serviceMap = com.salesmanager.core.util.ShippingUtil
					.buildServiceMap("upsxml", locale);

			*//** Details on whit RT quote information to display **//*
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
			}*/

			List<ShippingOption> shippingOptions = parsed.getOptions();
			if(shippingOptions!=null) {
				Map<String,String> details = module.getDetails();
				for(ShippingOption option : shippingOptions) {
					String name = details.get(option.getOptionCode());
					option.setOptionName(name);
					if(option.getOptionPrice()==null) {
						String priceText = option.getOptionPriceText();
						if(StringUtils.isBlank(priceText)) {
							throw new IntegrationException("Price text is null for option " + name);
						}
						try {
							BigDecimal price = new BigDecimal(priceText);
							option.setOptionPrice(price);
						} catch(Exception e) {
							throw new IntegrationException("Can't convert to numeric price " + priceText);
						}
					}
				}
			}
/*			if (options != null) {

				Map selectedintlservices = (Map) config
						.getConfiguration("service-global-upsxml");

				Iterator i = options.iterator();
				while (i.hasNext()) {
					ShippingOption option = (ShippingOption) i.next();
					// option.setCurrency(store.getCurrency());
					StringBuffer description = new StringBuffer();

					String code = option.getOptionCode();
					option.setOptionCode(code);
					// get description
					String label = (String) serviceMap.get(code);
					if (label == null) {
						log
								.warn("UPSXML cannot find description for service code "
										+ code);
					}

					option.setOptionName(label);

					description.append(option.getOptionName());
					if (displayQuoteDeliveryTime == ShippingConstants.DISPLAY_RT_QUOTE_TIME) {
						if (!StringUtils.isBlank(option
								.getEstimatedNumberOfDays())) {
							description.append(" (").append(
									option.getEstimatedNumberOfDays()).append(
									" ").append(
									labelUtil.getText(locale,
											"label.generic.days.lowercase"))
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

					if (!selectedintlservices.containsKey(option
							.getOptionCode())) {
						if (returnColl == null) {
							returnColl = new ArrayList();
						}
						returnColl.add(option);
						// options.remove(option);
					}

				}

				if (options.size() == 0) {
					LogMerchantUtil
							.log(
									store.getMerchantId(),
									" none of the service code returned by UPS ["
											+ selectedintlservices
													.keySet()
													.toArray(
															new String[selectedintlservices
																	.size()])
											+ "] for this shipping is in your selection list");
				}
			}*/



			return shippingOptions;
	}
		} catch (Exception e1) {
			LOGGER.error("UPS quote error",e1);
			throw new IntegrationException(e1);
		} finally {
			if (reader != null) {
				try {
					reader.close();
				} catch (Exception ignore) {
				}
			}

			if (httppost != null) {
				httppost.releaseConnection();
			}
		}
}


	@Override
	public CustomIntegrationConfiguration getCustomModuleConfiguration(
			MerchantStore store) throws IntegrationException {
		//nothing to do
		return null;
	}}


class UPSParsedElements  {

	private String statusCode;
	private String statusMessage;
	private String error = "";
	private String errorCode = "";
	private List<ShippingOption> options = new ArrayList<ShippingOption>();

	public void addOption(ShippingOption option) {
		options.add(option);
	}

	public List<ShippingOption> getOptions() {
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
