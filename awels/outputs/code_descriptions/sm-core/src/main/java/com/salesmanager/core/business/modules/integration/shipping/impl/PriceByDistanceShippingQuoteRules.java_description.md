# PriceByDistanceShippingQuoteRules.java

## Review

## 1. Summary
The `PriceByDistanceShippingQuoteRules` class is an implementation of the `ShippingQuoteModule` interface that calculates shipping options based on the distance between a store and a delivery address.  
- **Core idea**: retrieve the pre‑calculated distance (in km) from a `ShippingQuote`’s `quoteInformations` map, apply a simple per‑km rate, and return a single `ShippingOption`.  
- **Key components**:  
  * `getShippingQuotes` – the only method that performs business logic.  
  * `MODULE_CODE` – constant identifying this module.  
  * Unused stubs (`validateModuleConfiguration`, `getCustomModuleConfiguration`) – placeholder for future configuration logic.  
- **Design patterns / frameworks**: follows the *Strategy* pattern (different quote modules can be swapped). It relies on Apache Commons (`Validate`, `StringUtils`) and SLF4J for logging (currently unused).

## 2. Detailed Description
1. **Input validation**  
   - The method uses `Validate.notNull` / `notEmpty` to ensure `delivery`, `delivery.getCountry()`, and `packages` are provided.  
   - If the postal code is blank, the method returns `null` (indicating “no quote available”).

2. **Distance extraction**  
   - Attempts to read the distance from `quote.getQuoteInformations()` using the key `Constants.DISTANCE_KEY`.  
   - If the key is missing or the value is not a `Double`, the method returns `null`.

3. **Business rules**  
   - **Maximum distance** – hard‑coded to 150 km. Any distance above this threshold results in `null`.  
   - **Pricing** – a flat rate of 2 €/km for distances ≤ 20 km, 3 €/km otherwise.  
   - **Minimum distance** – intended to be 1 km, but applied *after* the total price calculation, so it has no effect.

4. **Option creation**  
   - Builds a `ShippingOption` setting only the price, module code, option code, and ID.  
   - Adds it to the quote’s option list (creating the list if necessary) and returns the list.

5. **Return value**  
   - Returns the list of options or `null` when the module decides no quote should be offered.  
   - No cleanup is required – all work is done in memory.

**Assumptions & constraints**  
- A distance value has already been calculated by another pre‑processor and stored under `Constants.DISTANCE_KEY`.  
- No handling of negative or zero distances.  
- All configuration (max distance, per‑km rates, etc.) is hard‑coded; no admin‑configurable parameters yet.  
- The method expects the caller to interpret `null` as “no shipping options”.

**Architectural choice**  
The module follows a simple “pull” model: it only reacts when a distance is present. This keeps the shipping pipeline decoupled – the distance calculation can be replaced or extended without touching this module.

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `validateModuleConfiguration(IntegrationConfiguration, MerchantStore)` | Stub – intended to validate configuration. Currently no-op. | Configuration, store | None | None |
| `getCustomModuleConfiguration(MerchantStore)` | Stub – intended to provide module‑specific config. Returns `null`. | Store | `CustomIntegrationConfiguration` (always `null`) | None |
| `getShippingQuotes(ShippingQuote, List<PackageDetails>, BigDecimal, Delivery, ShippingOrigin, MerchantStore, IntegrationConfiguration, IntegrationModule, ShippingConfiguration, Locale)` | Calculates shipping options based on distance. | Quote, packages, order total, delivery, origin, store, configuration, module, shipping config, locale | `List<ShippingOption>` or `null` | Creates/updates the quote’s options list |

### Notes on `getShippingQuotes`
- **Input validation**: uses Apache Commons `Validate`.
- **Distance extraction**: `Objects.nonNull` + map lookup.
- **Price calculation**: `total = new BigDecimal(distance).multiply(price)`.
- **Option creation**: populates only price and identifiers.

## 4. Dependencies
| Dependency | Type | Role |
|------------|------|------|
| `org.apache.commons.lang3.StringUtils` | Third‑party | Checks postal code string. |
| `org.apache.commons.lang3.Validate` | Third‑party | Defensive null / empty checks. |
| `org.slf4j.Logger` & `LoggerFactory` | Third‑party | Logging (currently unused). |
| `com.salesmanager.core.model.*` | Internal | Domain entities (`ShippingQuote`, `ShippingOption`, etc.). |
| `com.salesmanager.core.modules.constants.Constants` | Internal | Key for distance information. |
| `com.salesmanager.core.modules.integration.*` | Internal | Interfaces and exception handling. |

No platform‑specific or OS‑dependent libraries are used. The code relies on the Sales‑Manager core model classes.

## 5. Additional Notes

### Edge Cases & Potential Bugs
1. **Distance type mismatch** – the code casts the stored value to `Double`; if a different numeric type is stored (e.g., `Float`, `BigDecimal`), a `ClassCastException` will be thrown.  
2. **Negative or zero distance** – no checks; negative values would produce a negative price.  
3. **Minimum distance logic** – `distance < 1` is corrected *after* the total is computed, so the final price still reflects the original (smaller) distance.  
4. **Returning `null`** – callers need to treat a `null` result as “no shipping option”. Returning an empty list could be clearer and safer.  
5. **Hard‑coded parameters** – max distance, per‑km rates, and the “20 km split” are hard‑coded; configuration through the admin panel would be preferable.  

### Suggested Enhancements
- **Configuration injection**: expose max distance, rates, and thresholds via `IntegrationConfiguration` or a dedicated config class.  
- **Robust distance extraction**: use `Number` and `doubleValue()` or parse via `BigDecimal` to avoid casting issues.  
- **Validate distance range**: reject negative or zero distances early.  
- **Apply minimum distance before price calculation**.  
- **Return empty list instead of `null`** when no options are available.  
- **Populate additional `ShippingOption` fields** (e.g., name, description) for better UI display.  
- **Add logging** for key decision points (e.g., distance thresholds, pricing applied).  
- **Unit tests** covering all branches: distance <1, 1–20, >20, >150, missing distance key, negative distance, postal code blank, etc.

### Design Consistency
- The class implements a *Strategy* interface but does not use the provided configuration or module parameters.  
- Consistency with other shipping modules (e.g., those that might use `quote.getQuoteInformations()` or rely on external services) would be improved by centralizing configuration handling.

### Code Quality
- **Unused imports**: `Logger` and `CustomIntegrationConfiguration` are unused – clean up to avoid clutter.  
- **Javadoc**: Only a class-level comment exists; method-level Javadoc would aid maintainability.  
- **Magic numbers**: Replace literal distances and rates with named constants or config values.  

Overall, the implementation achieves its basic purpose but requires refinement to be production‑ready, especially regarding robustness, configurability, and adherence to best practices.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Objects;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingOption;
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.system.CustomIntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.modules.constants.Constants;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuoteModule;


/**
 * Requires to set pre-processor distance calculator
 * pre-processor calculates the distance (in kilometers [can be changed to miles]) based on delivery address
 * when that module is invoked during process it will calculate the price
 * DISTANCE * PRICE/KM
 * @author carlsamson
 *
 */
public class PriceByDistanceShippingQuoteRules implements ShippingQuoteModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(PriceByDistanceShippingQuoteRules.class);

	public final static String MODULE_CODE = "priceByDistance";

	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		// Not used

	}

	@Override
	public CustomIntegrationConfiguration getCustomModuleConfiguration(
			MerchantStore store) throws IntegrationException {
		// Not used
		return null;
	}

	@Override
	public List<ShippingOption> getShippingQuotes(ShippingQuote quote,
			List<PackageDetails> packages, BigDecimal orderTotal,
			Delivery delivery, ShippingOrigin origin, MerchantStore store,
			IntegrationConfiguration configuration, IntegrationModule module,
			ShippingConfiguration shippingConfiguration, Locale locale)
			throws IntegrationException {

		
		
		Validate.notNull(delivery, "Delivery cannot be null");
		Validate.notNull(delivery.getCountry(), "Delivery.country cannot be null");
		Validate.notNull(packages, "packages cannot be null");
		Validate.notEmpty(packages, "packages cannot be empty");
		
		//requires the postal code
		if(StringUtils.isBlank(delivery.getPostalCode())) {
			return null;
		}

		Double distance = null;

		//look if distance has been calculated
		if (Objects.nonNull(quote) && Objects.nonNull(quote.getQuoteInformations())) {
			if (quote.getQuoteInformations().containsKey(Constants.DISTANCE_KEY)) {
				distance = (Double) quote.getQuoteInformations().get(Constants.DISTANCE_KEY);
			}
		}
		
		if(distance==null) {
			return null;
		}
		
		//maximum distance TODO configure from admin
		if(distance > 150D) {
			return null;
		}
		
		List<ShippingOption> options = quote.getShippingOptions();
		
		if(options == null) {
			options = new ArrayList<>();
			quote.setShippingOptions(options);
		}
		
		BigDecimal price = null;
		
		if(distance<=20) {
			price = new BigDecimal(2);//TODO from the admin
		} else {
			price = new BigDecimal(3);//TODO from the admin
		}
		BigDecimal total = new BigDecimal(distance).multiply(price);
		
		if(distance < 1) { //minimum 1 unit
			distance = 1D;
		}


		ShippingOption shippingOption = new ShippingOption();
			
			
		shippingOption.setOptionPrice(total);
		shippingOption.setShippingModuleCode(MODULE_CODE);
		shippingOption.setOptionCode(MODULE_CODE);
		shippingOption.setOptionId(MODULE_CODE);

		options.add(shippingOption);

		
		return options;
		
		
	}

}


```
