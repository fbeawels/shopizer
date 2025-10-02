# ShippingDistancePreProcessorImpl.java

## Review

## 1. Summary  
**Purpose** – `ShippingDistancePreProcessorImpl` is a Spring‐managed component that enriches a `ShippingQuote` with geocoding and distance data derived from the Google Maps API.  
**Key Responsibilities**  
1. **Zone filtering** – Only processes quotes whose delivery zone matches a configurable list of “allowed” zone codes.  
2. **Geocoding** – Builds full origin and destination addresses, resolves them to latitude/longitude pairs using `GeocodingApi`.  
3. **Distance matrix** – Requests a straight‑line distance (in metres) between origin and destination via `DistanceMatrixApi`, converts it to kilometres and stores it in the quote’s `quoteInformations` map under `Constants.DISTANCE_KEY`.  
4. **Metadata** – Persists the destination latitude/longitude on the `Delivery` instance for later use (e.g., rendering a map).  

**Notable Design Choices**  
- Uses **Spring’s `@Component`** to integrate with the larger application.  
- Leverages **Apache Commons** (`StringUtils`, `Validate`) for string handling and argument validation.  
- Depends on the **Google Maps Services Java client** for all geospatial lookups.  

---

## 2. Detailed Description  
### Initialization  
- The component is instantiated by Spring; the `apiKey` and `allowedZonesCodes` are injected from the application’s property file.  
- No explicit initialization logic; each call to `prePostProcessShippingQuotes` constructs a new `GeoApiContext`.

### Runtime Flow  
1. **Early Exit Conditions**  
   - If the delivery has no zone → return.  
   - If the zone is not in the `allowedZonesCodes` list → return.  
   - If the delivery postal code is blank → return.  
   - If the API key is null → `Validate.notNull` throws an exception that is later caught and logged.

2. **Address Construction**  
   - Origin and destination addresses are built by concatenating address lines, city, postal code, state (if present), zone code, and country ISO code.  
   - The builder pattern uses a simple `StringBuilder` with spaces as separators.

3. **Geocoding**  
   - The two full addresses are sent to `GeocodingApi.geocode()`; the first result of each call provides the latitude/longitude.  
   - Destination coordinates are stored back onto the `Delivery` object.

4. **Distance Calculation**  
   - A `DistanceMatrixRequest` is built with the two coordinate pairs.  
   - `awaitIgnoreError()` is used so that any HTTP errors return `null`.  
   - If a matrix is returned, the first element’s distance (in metres) is extracted and converted to kilometres (multiplied by `0.001`).  
   - The value is placed into `quote.getQuoteInformations()` under `Constants.DISTANCE_KEY`.  
   - If the matrix is `null` or any array access fails, a warning is logged.

5. **Exception Handling**  
   - All exceptions (including `IllegalArgumentException` from `Validate.notNull`) are caught, logged, and swallowed.  
   - No `IntegrationException` is thrown; callers cannot distinguish failures.

### Cleanup  
- The component holds no resources that require explicit cleanup.  
- The `GeoApiContext` is not closed; the Google client manages its own connection pool.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `prePostProcessShippingQuotes` | Main hook that enriches a `ShippingQuote` with distance and geocode data. | `ShippingQuote quote, List<PackageDetails> packages, BigDecimal orderTotal, Delivery delivery, ShippingOrigin origin, MerchantStore store, IntegrationConfiguration globalShippingConfiguration, IntegrationModule currentModule, ShippingConfiguration shippingConfiguration, List<IntegrationModule> allModules, Locale locale` | `void` | Modifies `quote` (adds distance key), sets `delivery` latitude/longitude, logs errors. |
| `getAllowedZonesCodes` | Getter for the list of zone codes that this module will accept. | None | `List<String>` | None |
| `setAllowedZonesCodes` | Setter for the allowed zone codes (used by Spring or manually). | `List<String> allowedZonesCodes` | `void` | Replaces internal list. |
| `getModuleCode` | Returns the unique code for this module. | None | `String` (`"shippingDistanceModule"`) | None |

*Utility*: The class relies on Apache Commons’ `StringUtils.isBlank` and `Validate.notNull` for argument checks and string validation.

---

## 4. Dependencies  

| Library | Role | Third‑Party? | Notes |
|---------|------|--------------|-------|
| **Spring Framework** (`@Component`, `@Value`) | Dependency injection & bean lifecycle | Third‑Party | Required for configuration values. |
| **Google Maps Services Java client** (`GeoApiContext`, `GeocodingApi`, `DistanceMatrixApi`) | Geocoding & distance calculations | Third‑Party | Must be added to Maven/Gradle; requires a valid API key. |
| **Apache Commons Lang** (`StringUtils`, `Validate`) | String utilities & validation | Third‑Party | Simple string checks; could be replaced by Java 11+ `String.isBlank()`. |
| **SLF4J** (`Logger`, `LoggerFactory`) | Logging | Third‑Party | Standard logging abstraction. |
| **SalesManager domain models** (`Delivery`, `ShippingQuote`, etc.) | Domain objects | Internal | Provided by the project’s core module. |
| **Constants** (`Constants.DISTANCE_KEY`) | Key for quote information map | Internal | Likely a static string. |

No platform‑specific APIs are used; the code is portable across any JVM environment with the required libraries.

---

## 5. Additional Notes  

### Strengths  
- **Clear separation of concerns** – the pre‑processor only enriches data; pricing logic is elsewhere.  
- **Simple configuration** – zone filtering and API key are externalised.  
- **Use of existing robust libraries** – leverages well‑maintained Google and Commons APIs.

### Weaknesses & Edge Cases  
1. **Error handling** – Swallowing all exceptions hides failures from callers. A missing API key or quota exhaustion will silently produce incomplete quotes. Consider rethrowing a checked `IntegrationException`.  
2. **Null / Empty Address Parts** – `origin.getAddress()` or other fields could be `null`, leading to the string `"null"` in the query. Pre‑validate each component.  
3. **Array Bounds** – After a successful geocoding request the code assumes `rows[0]` and `elements[0]` exist. A malformed response could cause `ArrayIndexOutOfBoundsException`. Add defensive checks.  
4. **`allowedZonesCodes` defaulting to `null`** – If not configured, the processor will reject *all* zones. It may be more intuitive to default to “accept all zones” or throw a configuration error.  
5. **Distance unit conversion** – Uses `0.001 * distance.inMeters`. If the distance is large, a `double` may lose precision. Consider using `BigDecimal` or storing metres directly.  
6. **Re‑use of `GeoApiContext`** – Constructing a new context for every call is costly. A shared, thread‑safe instance could be injected.  
7. **Logging** – Errors are logged at `error` level but the message often contains generic wording (“Exception while calculating the shipping distance”). Including the exception type and stack trace (already done) is good, but a more descriptive message could aid debugging.  
8. **Locale/Internationalization** – The component ignores `Locale`; if different locales require different address formatting, the code should adapt accordingly.  

### Suggested Enhancements  
- **Fail‑fast configuration validation** – validate API key and zone list at startup.  
- **Return a meaningful status** – wrap results in a custom result object or propagate an `IntegrationException`.  
- **Input sanitization** – trim fields and reject addresses that are too short.  
- **Unit tests** – mock the Google API to test address building, distance extraction, and failure paths.  
- **Caching** – cache geocoding results for repeated addresses to reduce API calls.  
- **Metric instrumentation** – track number of successful/failed distance lookups for observability.  

Overall, the component delivers the intended functionality but would benefit from more robust error handling, defensive programming, and configurability to improve reliability in production deployments.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.math.BigDecimal;
import java.util.List;
import java.util.Locale;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import com.google.maps.DistanceMatrixApi;
import com.google.maps.GeoApiContext;
import com.google.maps.GeocodingApi;
import com.google.maps.model.Distance;
import com.google.maps.model.DistanceMatrix;
import com.google.maps.model.DistanceMatrixRow;
import com.google.maps.model.GeocodingResult;
import com.google.maps.model.LatLng;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.modules.constants.Constants;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuotePrePostProcessModule;

/**
 * Uses google api to get lng, lat and distance in km for a given delivery address
 * The route will be displayed on a map to the end user and available
 * from the admin section
 * 
 * The module can be configured to use miles by changing distance.inMeters
 * 
 * To use this pre-processor you will need a google api-key
 * 
 * Access google developers console
 * https://console.developers.google.com/project
 * 
 * Geocoding
 * Distance Matrix
 * Directions
 * 
 * Create new key for server application
 * Copy API key
 * https://console.developers.google.com
 * 
 * https://developers.google.com/maps/documentation/webservices/client-library
 * https://github.com/googlemaps/google-maps-services-java/tree/master/src/test/java/com/google/maps
 * 
 * @author carlsamson
 *
 */
@Component("shippingDistancePreProcessor")
public class ShippingDistancePreProcessorImpl implements ShippingQuotePrePostProcessModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ShippingDistancePreProcessorImpl.class);
	
	private final static String BLANK = " ";
	
	private final static String MODULE_CODE = "shippingDistanceModule";

	@Value("${config.shippingDistancePreProcessor.apiKey}")
	private String apiKey;

	@Value("#{'${config.shippingDistancePreProcessor.acceptedZones}'.split(',')}") 
	private List<String> allowedZonesCodes = null;



	public List<String> getAllowedZonesCodes() {
		return allowedZonesCodes;
	}





	public void setAllowedZonesCodes(List<String> allowedZonesCodes) {
		this.allowedZonesCodes = allowedZonesCodes;
	}





	public void prePostProcessShippingQuotes(ShippingQuote quote,
			List<PackageDetails> packages, BigDecimal orderTotal,
			Delivery delivery, ShippingOrigin origin, MerchantStore store,
			IntegrationConfiguration globalShippingConfiguration,
			IntegrationModule currentModule,
			ShippingConfiguration shippingConfiguration,
			List<IntegrationModule> allModules, Locale locale)
			throws IntegrationException {
		
		
		/** which destinations are supported by this module **/
		
		if(delivery.getZone()==null) {
			return;
		}
		
		boolean zoneAllowed = false;
		if(allowedZonesCodes!=null) {
			for(String zoneCode : allowedZonesCodes) {
				if(zoneCode.equals(delivery.getZone().getCode())) {
					zoneAllowed = true;
					break;
				}
			}
		}
		
		if(!zoneAllowed) {
			return;
		}
		
		if(StringUtils.isBlank(delivery.getPostalCode())) {
			return;
		}
		
		Validate.notNull(apiKey, "Requires the configuration of google apiKey");
		
		GeoApiContext context = new GeoApiContext().setApiKey(apiKey);
		
		//build origin address
		StringBuilder originAddress = new StringBuilder();
		
		originAddress.append(origin.getAddress()).append(BLANK)
		.append(origin.getCity()).append(BLANK)
		.append(origin.getPostalCode()).append(BLANK);
		
		if(!StringUtils.isBlank(origin.getState())) {
			originAddress.append(origin.getState()).append(" ");
		}
		if(origin.getZone()!=null) {
			originAddress.append(origin.getZone().getCode()).append(" ");
		}
		originAddress.append(origin.getCountry().getIsoCode());

		
		//build destination address
		StringBuilder destinationAddress = new StringBuilder();
		
		destinationAddress.append(delivery.getAddress()).append(BLANK);
		if(!StringUtils.isBlank(delivery.getCity())) {
			destinationAddress.append(delivery.getCity()).append(BLANK);
		}
		destinationAddress.append(delivery.getPostalCode()).append(BLANK);
		
		if(!StringUtils.isBlank(delivery.getState())) {
			destinationAddress.append(delivery.getState()).append(" ");
		}
		if(delivery.getZone()!=null) {
			destinationAddress.append(delivery.getZone().getCode()).append(" ");
		}
		destinationAddress.append(delivery.getCountry().getIsoCode());
		
		
		try {
			GeocodingResult[] originAdressResult =  GeocodingApi.geocode(context,
					originAddress.toString()).await();

			GeocodingResult[] destinationAdressResult =  GeocodingApi.geocode(context,
					destinationAddress.toString()).await();

			if(originAdressResult.length>0 && destinationAdressResult.length>0) {
				LatLng originLatLng = originAdressResult[0].geometry.location;
				LatLng destinationLatLng = destinationAdressResult[0].geometry.location;
				
				delivery.setLatitude(String.valueOf(destinationLatLng.lat));
				delivery.setLongitude(String.valueOf(destinationLatLng.lng));
				
				//keep latlng for further usage in order to display the map
	
				
				DistanceMatrix  distanceRequest = DistanceMatrixApi.newRequest(context)
		 		.origins(new LatLng(originLatLng.lat, originLatLng.lng))
		 		
		 		.destinations(new LatLng(destinationLatLng.lat, destinationLatLng.lng))
		 				.awaitIgnoreError();
				
				
				if(distanceRequest!=null) {
					DistanceMatrixRow distanceMax = distanceRequest.rows[0];
					Distance distance = distanceMax.elements[0].distance;
					quote.getQuoteInformations().put(Constants.DISTANCE_KEY, 0.001 * distance.inMeters);
				} else {
				  LOGGER.error("Expected distance inner google api to return DistanceMatrix, it returned null. API key might not be working for this request");
				}

			}
		
		} catch (Exception e) {
			LOGGER.error("Exception while calculating the shipping distance",e);
		}

	}




	
	public String getModuleCode() {
		return MODULE_CODE;
	}














}



```
