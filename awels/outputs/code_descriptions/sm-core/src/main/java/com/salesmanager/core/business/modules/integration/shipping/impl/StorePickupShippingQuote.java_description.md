# StorePickupShippingQuote.java

## Review

## 1. Summary  

**Purpose**  
The class `StorePickupShippingQuote` implements a shipping module that adds a “Store Pick‑Up” option to a shipping quote.  
* The module is identified by the code `storePickUp`.  
* It validates that a price and a note are configured.  
* During the quote‑generation pipeline it injects a single `ShippingOption` whose price comes from the module’s configuration and whose identifier is built from the delivery region.

**Key Components**  

| Component | Role |
|-----------|------|
| `MODULE_CODE` | Constant that identifies the module. |
| `merchantConfigurationService` | Service used to fetch configuration values (unused in the current implementation). |
| `productPriceUtils` | Utility that converts price strings to `BigDecimal` and formats them for display. |
| `validateModuleConfiguration` | Ensures the required integration keys exist and are syntactically correct. |
| `prePostProcessShippingQuotes` | Hooks into the shipping quote pipeline to inject the pickup option. |
| `getShippingQuotes` / `getCustomModuleConfiguration` | Stubs – the module does not provide its own quote generation logic. |

The class implements two interfaces:  

* `ShippingQuoteModule` – defines the basic contract for a shipping module.  
* `ShippingQuotePrePostProcessModule` – allows the module to modify the quote after all quotes have been collected.

**Design Patterns / Frameworks**  
* **Dependency Injection** – `@Inject` annotations suggest that a CDI or Spring‑style container is used.  
* **Strategy/Plugin** – Shipping modules are plugged into a larger shipping framework via the two interfaces.  
* **Utility Class** – `ProductPriceUtils` provides helper methods for monetary values.

---

## 2. Detailed Description  

### Execution Flow  

1. **Initialization**  
   * The container injects `merchantConfigurationService` and `productPriceUtils`.  
   * No explicit constructor logic – all work is performed in the interface callbacks.

2. **Configuration Validation**  
   * When a store configuration is saved or edited, `validateModuleConfiguration` is invoked.  
   * It checks that the `integrationConfiguration` contains a non‑null `price` key that can be parsed to `BigDecimal`, and that a `note` key exists.  
   * If any checks fail, an `IntegrationException` with the offending field(s) is thrown.

3. **Quote Generation Pipeline**  
   * Shipping modules are invoked in two phases:  
     * **Collecting** – `getShippingQuotes` (not used here).  
     * **Post‑processing** – `prePostProcessShippingQuotes`.  
   * The post‑processing step runs **after** the system has gathered all standard shipping options.  
   * It reads the configured price and region, creates a `ShippingOption` and attaches it to the quote.

4. **Cleanup**  
   * No explicit cleanup; the object is stateless, so no resources need releasing.

### Core Interaction  

| Method | Input | Operation | Output |
|--------|-------|-----------|--------|
| `validateModuleConfiguration` | `IntegrationConfiguration`, `MerchantStore` | Checks keys and value formats | Throws `IntegrationException` on failure |
| `prePostProcessShippingQuotes` | `ShippingQuote`, `List<PackageDetails>`, … | Creates/attaches a pickup option | Mutates `ShippingQuote` |
| `getShippingQuotes` | *unused* | **Not implemented** | Returns `null` (acceptable for this module) |
| `getCustomModuleConfiguration` | `MerchantStore` | **Not implemented** | Returns `null` |

### Assumptions & Constraints  

* The module assumes that the integration configuration will contain a `price` key that is a valid decimal string and a `note` key (currently only checked for existence, not blankness).  
* It relies on the container to inject dependencies.  
* The `price` is treated as a fixed value per store; no per‑delivery‑zone pricing is supported.  
* `productPriceUtils.getAmount(String)` is expected to throw a runtime exception on bad input – this is wrapped into an `IntegrationException`.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `validateModuleConfiguration` | Validates integration keys for price and note | `integrationConfiguration`, `store` | None (throws exception on error) | May throw `IntegrationException` |
| `getShippingQuotes` | (Not used) Stub for generating quotes | See signature | `null` | None |
| `getCustomModuleConfiguration` | (Not used) Stub for custom config | `store` | `null` | None |
| `prePostProcessShippingQuotes` | Adds store‑pickup option to the quote | `quote`, `packages`, `orderTotal`, `delivery`, `origin`, `store`, `globalShippingConfiguration`, `currentModule`, `shippingConfiguration`, `allModules`, `locale` | None | Mutates `quote` (adds option, sets default if none selected) |
| `getModuleCode` | Returns module identifier | None | `MODULE_CODE` | None |

**Reusable/Utility Methods**  
* The class itself does not expose reusable utilities, but it delegates price parsing/formatting to `productPriceUtils`.  
* `Validate.notNull` from Apache Commons is used for guard clauses.

---

## 4. Dependencies  

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `javax.inject.Inject` | CDI/Spring injection | Standard dependency‑injection annotation |
| `org.apache.commons.lang3.Validate` | Apache Commons | Provides guard clauses |
| `com.salesmanager.core.business.services.system.MerchantConfigurationService` | In‑house service | Not used in the current logic |
| `com.salesmanager.core.business.utils.ProductPriceUtils` | In‑house utility | Converts and formats monetary amounts |
| `com.salesmanager.core.model.*` | Domain entities | ShippingOption, ShippingQuote, etc. |
| `com.salesmanager.core.modules.integration.*` | In‑house integration framework | Defines module interfaces, exception |
| `java.math.BigDecimal`, `java.util.*`, `java.util.Locale` | JDK | Standard libraries |

No external, non‑standard libraries beyond Apache Commons and the Sales Manager core modules.

---

## 5. Additional Notes & Recommendations  

### Validation Issues  
* **Note Key Check** – The current logic only adds `"note"` to `errorFields` when `keys` is `null`. If `keys` is present but the `"note"` key is missing or blank, the error is silently ignored.  
  *Recommendation:* Use `StringUtils.isBlank(keys.get("note"))` (from Apache Commons Lang) to detect missing or blank notes.  
* **Price Parsing** – `new BigDecimal(keys.get("price"))` will throw a `NullPointerException` if `"price"` is absent.  
  *Recommendation:* Guard against `null` before parsing, or use a helper method that returns a boolean flag.

### Code Duplication & Clarity  
* The error‑collection logic creates a new list each time an error is found, potentially discarding earlier errors.  
  *Recommendation:* Instantiate `errorFields` once and add all offending keys.  
* The module’s code is a bit sparse; consider adding JavaDoc for public methods, especially to clarify that the module only works in a post‑processing phase.

### Robustness  
* **Null Checks** – `delivery.getZone()` or `delivery.getState()` might be `null`. The current implementation assumes at least one is non‑null. Add defensive checks or default region handling.  
* **Active Check** – The method returns early if `globalShippingConfiguration.isActive()` is `false`. This is fine, but consider logging a debug message to aid troubleshooting.  
* **Locale Handling** – `productPriceUtils.getStoreFormatedAmountWithCurrency` uses the `store`’s currency; ensure that the store always has a currency set.

### Future Enhancements  
1. **Per‑Zone Pricing** – Allow configuration of different prices per zone, e.g., a map `price_by_zone`.  
2. **Custom Message** – Expose the configured `note` in the `ShippingOption` so that the front‑end can display it to the customer.  
3. **Unit Tests** – Add comprehensive tests covering validation, price parsing, and option creation.  
4. **Logging** – Integrate a logger to trace when options are added or when validation fails.  

### Architectural Notes  
* The class relies heavily on side‑effects (mutating the passed `ShippingQuote`). This is typical for the integration framework but can be error‑prone if other modules perform similar mutations. A clearer contract or immutable objects would reduce accidental overwrites.  
* If the surrounding system evolves to support multiple pickup locations or dynamic pricing, the current single‑price approach will need refactoring.

---

**Overall Assessment**  
The implementation fulfills its basic contract: it validates configuration and injects a pickup shipping option. The code is straightforward and readable, but it contains a few oversight issues in validation and defensive programming. Addressing these will improve reliability, maintainability, and testability of the module.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.inject.Inject;

import org.apache.commons.lang3.Validate;

import com.salesmanager.core.business.services.system.MerchantConfigurationService;
import com.salesmanager.core.business.utils.ProductPriceUtils;
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
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuoteModule;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuotePrePostProcessModule;


/**
 * Store pick up shipping module
 * 
 * Requires a configuration of a message note to be printed to the client
 * and a price for calculation (should be configured to 0)
 * 
 * Calculates a ShippingQuote with a price set to the price configured
 * @author carlsamson
 *
 */
public class StorePickupShippingQuote implements ShippingQuoteModule, ShippingQuotePrePostProcessModule {
	
	
	public final static String MODULE_CODE = "storePickUp";

	@Inject
	private MerchantConfigurationService merchantConfigurationService;
	
	@Inject
	private ProductPriceUtils productPriceUtils;


	@Override
	public void validateModuleConfiguration(
			IntegrationConfiguration integrationConfiguration,
			MerchantStore store) throws IntegrationException {
		
		
		
		
		List<String> errorFields = null;
		
		//validate integrationKeys['account']
		Map<String,String> keys = integrationConfiguration.getIntegrationKeys();
		//if(keys==null || StringUtils.isBlank(keys.get("price"))) {
		if(keys==null) {
			errorFields = new ArrayList<String>();
			errorFields.add("price");
		} else {
			//validate it can be parsed to BigDecimal
			try {
				BigDecimal price = new BigDecimal(keys.get("price"));
			} catch(Exception e) {
				errorFields = new ArrayList<String>();
				errorFields.add("price");
			}
		}
		
		//if(keys==null || StringUtils.isBlank(keys.get("note"))) {
		if(keys==null) {
			errorFields = new ArrayList<String>();
			errorFields.add("note");
		}


		
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

		// TODO Auto-generated method stub
		return null;

	}

	@Override
	public CustomIntegrationConfiguration getCustomModuleConfiguration(
			MerchantStore store) throws IntegrationException {
		// TODO Auto-generated method stub
		return null;
	}

	@Override
	public void prePostProcessShippingQuotes(ShippingQuote quote,
			List<PackageDetails> packages, BigDecimal orderTotal,
			Delivery delivery, ShippingOrigin origin, MerchantStore store,
			IntegrationConfiguration globalShippingConfiguration,
			IntegrationModule currentModule,
			ShippingConfiguration shippingConfiguration,
			List<IntegrationModule> allModules, Locale locale)
			throws IntegrationException {
		
		Validate.notNull(globalShippingConfiguration, "IntegrationConfiguration must not be null for StorePickUp");
		
		
		try {
			
			if(!globalShippingConfiguration.isActive())
				return;

			String region = null;
			
			String price = globalShippingConfiguration.getIntegrationKeys().get("price");
	
	
			if(delivery.getZone()!=null) {
				region = delivery.getZone().getCode();
			} else {
				region = delivery.getState();
			}
			
			ShippingOption shippingOption = new ShippingOption();
			shippingOption.setShippingModuleCode(MODULE_CODE);
			shippingOption.setOptionCode(MODULE_CODE);
			shippingOption.setOptionId(new StringBuilder().append(MODULE_CODE).append("_").append(region).toString());
			
			shippingOption.setOptionPrice(productPriceUtils.getAmount(price));
	
			shippingOption.setOptionPriceText(productPriceUtils.getStoreFormatedAmountWithCurrency(store, productPriceUtils.getAmount(price)));
	
			List<ShippingOption> options = quote.getShippingOptions();
			
			if(options == null) {
				options = new ArrayList<ShippingOption>();
				quote.setShippingOptions(options);
			}

			options.add(shippingOption);
			
			if(quote.getSelectedShippingOption()==null) {
				quote.setSelectedShippingOption(shippingOption);
			}

		
		} catch (Exception e) {
			throw new IntegrationException(e);
		}
	
		
		
	}

	@Override
	public String getModuleCode() {
		// TODO Auto-generated method stub
		return MODULE_CODE;
	}



}



```
