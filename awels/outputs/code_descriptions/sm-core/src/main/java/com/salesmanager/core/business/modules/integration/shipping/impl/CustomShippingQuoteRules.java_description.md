# CustomShippingQuoteRules.java

## Review

## 1. Summary

**Purpose**  
`CustomShippingQuoteRules` is a Shipping‑Quote module that integrates with the Drools rule engine to calculate shipping options based on package dimensions, weight, distance, and destination. The module implements the `ShippingQuoteModule` interface so that it can be plugged into the larger SalesManager shipping workflow.

**Key Components**

| Component | Role |
|-----------|------|
| `getShippingQuotes` | Core routine that receives a `ShippingQuote`, list of `PackageDetails`, and shipping context, builds a `ShippingInputParameters` DTO, runs a Drools rule (`PriceByDistance.drl`) and returns a list of `ShippingOption`s. |
| `DroolsBeanFactory` | Helper that supplies a configured `KieSession` from the Drools runtime. |
| `ShippingInputParameters` & `DecisionResponse` | DTOs used to communicate with the rule file. |
| `MODULE_CODE` | Identifier for the shipping module that is added to the option objects. |

**Design notes**  
- The module follows the *Strategy* pattern: different shipping modules can be swapped in without touching the core logic.  
- Uses *Dependency Injection* (`@Inject`) for the `DroolsBeanFactory`.  
- The rule file is loaded from the class‑path (`ResourceFactory.newClassPathResource`).

---

## 2. Detailed Description

### Execution Flow

1. **Validation**  
   * Basic non‑null checks (`delivery`, `packages`, `delivery.country`).  
   * If the postal code is missing the method returns `null` – effectively saying “no shipping options”.

2. **Pre‑computation**  
   * **Distance** – extracted from the `quote`’s information map if present.  
   * **Weight, Volume, Largest Size** – iterated over each `PackageDetails`.  
   * These values are converted to the integer types expected by `ShippingInputParameters`.

3. **Input DTO**  
   * Instantiates `ShippingInputParameters` and populates:
     * `weight` (long) – total weight cast from double.  
     * `country` – ISO code.  
     * `province` – zone code if available, otherwise “\*”.  
     * `moduleName` – the code of the current module.  
     * `distance` – long representation of the distance.  
     * `volume` – largest package volume.

4. **Rule Execution**  
   * Obtains a `KieSession` from `DroolsBeanFactory` using the hard‑coded rule file.  
   * Inserts the input DTO, sets a global `decision` object, and fires all rules.  
   * The rule is expected to set `DecisionResponse.customPrice`.

5. **Result Mapping**  
   * If a price was produced, a new `ShippingOption` is created, priced with `BigDecimal`, and labeled with the module’s constants.  
   * The option is added to the quote’s options list, which is then returned.

6. **Cleanup** – none; the `KieSession` is discarded after use.

### Assumptions & Constraints

| Assumption | Impact |
|------------|--------|
| `quote.getQuoteInformations()` contains a `DISTANCE_KEY` key if distance has been computed elsewhere. | If missing, distance is ignored. |
| `ShippingInputParameters` expects integer values (long) – weight, volume, distance. | Precision is lost when casting from `double`. |
| The Drools rule file is located at `com/salesmanager/drools/rules/PriceByDistance.drl`. | If the path is wrong (typo vs. `com/salesmanager`), rule loading will fail silently and no options are returned. |
| A non‑null `IntegrationModule` is supplied (used only for its code). | No validation of `module`. |
| Postal code is required to calculate quotes. | In its absence, the method returns `null`. |
| Thread safety of `KieSession` is handled by creating a new session for every call. | No shared mutable state. |

### Architecture

The class is a **plain service** that relies on the **injection** of the rule engine factory. Its responsibilities are strictly limited to:

* Parameter validation,
* Input transformation,
* Rule execution,
* Mapping rule output to domain objects.

This separation keeps the business logic decoupled from the rule engine and from the rest of the system.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `validateModuleConfiguration(IntegrationConfiguration, MerchantStore)` | Stub – required by the interface. | `integrationConfiguration`, `store` | `void` | None |
| `getCustomModuleConfiguration(MerchantStore)` | Stub – required by the interface. | `store` | `CustomIntegrationConfiguration` (always `null`) | None |
| `getShippingQuotes(ShippingQuote, List<PackageDetails>, BigDecimal, Delivery, ShippingOrigin, MerchantStore, IntegrationConfiguration, IntegrationModule, ShippingConfiguration, Locale)` | Main routine that calculates shipping options. | As per signature | `List<ShippingOption>` | Adds an option to the provided `quote` if the rule produces a price. |
| `main` (none – this class is a Spring bean used by the framework). |

**Utility/Helper Methods** – none; all logic lives inside `getShippingQuotes`.

---

## 4. Dependencies

| Library / Framework | Version (implied) | Purpose |
|---------------------|-------------------|---------|
| `org.apache.commons:commons-lang3` | – | `StringUtils`, `Validate` |
| `org.slf4j:slf4j-api` | – | Logging |
| `org.kie.api` & `org.kie.internal` | – | Drools KIE runtime (`KieSession`, `ResourceFactory`) |
| `javax.inject` | – | Dependency injection (`@Inject`) |
| SalesManager core modules | – | Domain classes (`ShippingOption`, `PackageDetails`, etc.) |

All dependencies are **third‑party** and widely used. No platform‑specific assumptions beyond a JDK that supports CDI (or Spring) for injection.

---

## 5. Additional Notes & Recommendations

### 1. Rule File Path

The class references `"com/salesmanager/drools/rules/PriceByDistance.drl"`.  
The rest of the project uses the **`com.salesmanager`** namespace.  
If the rule file actually lives under `com/salesmanager/...`, the rule will never be found, leading to silent failures.  
**Fix:** Verify the package name and either correct the string or relocate the DRL file.

### 2. Input Precision Loss

Weight, volume, and distance are all `double` values that are cast to `long` before being sent to the rule.  
* This truncates fractional values, which may impact pricing accuracy.  
* Consider passing the raw `double` values or using `BigDecimal` in the DTO if the rule file expects decimals.

### 3. Error Handling & Logging

* The method returns `null` when the postal code is missing.  
  Returning an **empty list** (`Collections.emptyList()`) would be clearer to callers and avoid `NullPointerException`s.  
* If the Drools session fails to load or fire (e.g., rule file missing), the exception is swallowed and the method silently returns the pre‑existing options.  
  Adding logging in the catch block or re‑throwing a domain‑specific exception would aid debugging.

### 4. Thread Safety & Session Management

Creating a new `KieSession` per call is safe, but the `DroolsBeanFactory` may also hold shared state.  
* Ensure that `droolsBeanFactory.getKieSession(...)` is thread‑safe or use a stateless `StatelessKieSession` if available.

### 5. Documentation & Code Readability

* Add JavaDoc comments to the public methods, especially `getShippingQuotes`, describing the expected inputs and the format of the rule output.  
* Extract the “largest volume” and “largest size” calculations into private helper methods for readability.

### 6. Nullability & Defensive Coding

* The method accesses `quote.getShippingOptions()` without checking if `quote` itself is `null`.  
  While the signature suggests it will never be `null`, defensive coding can prevent accidental NPEs.  
* Similarly, the code assumes `delivery.getCountry()` is non‑null, but the earlier `Validate` check only ensures `delivery` and its `country` property. Adding a null‑check on the returned country would make the code safer.

### 7. Future Enhancements

* **Caching** – If the same inputs recur often, cache the `ShippingOption` results to avoid re‑executing the rule.  
* **Rule Configuration** – Expose rule file path and module code as configuration properties instead of hardcoding them.  
* **Unit Tests** – Provide tests that inject a mock `DroolsBeanFactory` to validate the mapping logic without needing a real rule file.  

---

**Verdict**  
The class fulfills its intended purpose of bridging the core shipping calculation logic with a Drools rule set. The code is straightforward and readable, but a few edge cases and potential misconfigurations could lead to silent failures. Addressing the issues above will improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Locale;

import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.kie.api.runtime.KieSession;
import org.kie.internal.io.ResourceFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.configuration.DroolsBeanFactory;
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


public class CustomShippingQuoteRules implements ShippingQuoteModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(CustomShippingQuoteRules.class);
	
	@Inject
	private DroolsBeanFactory droolsBeanFactory;

	public final static String MODULE_CODE = "customQuotesRules";

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
		
		if(quote!=null) {
			//look if distance has been calculated
			if(quote.getQuoteInformations()!=null) {
				if(quote.getQuoteInformations().containsKey(Constants.DISTANCE_KEY)) {
					distance = (Double)quote.getQuoteInformations().get(Constants.DISTANCE_KEY);
				}
			}
		}
		
		//calculate volume (L x W x H)
		Double volume = null;
		Double weight = 0D;
		Double size = null;
		//calculate weight
		for(PackageDetails pack : packages) {
			weight = weight + pack.getShippingWeight();
			Double tmpVolume = pack.getShippingHeight() * pack.getShippingLength() * pack.getShippingWidth();
			if(volume == null || tmpVolume > volume) { //take the largest volume
				volume = tmpVolume;
			} 
			//largest size
			List<Double> sizeList = new ArrayList<Double>();
			sizeList.add(pack.getShippingHeight());
			sizeList.add(pack.getShippingWidth());
			sizeList.add(pack.getShippingLength());
			Double maxSize = Collections.max(sizeList);
			if(size==null || maxSize > size) {
				size = maxSize;
			}
		}
		
		//Build a ShippingInputParameters
		ShippingInputParameters inputParameters = new ShippingInputParameters();
		
		inputParameters.setWeight((long)weight.doubleValue());
		inputParameters.setCountry(delivery.getCountry().getIsoCode());
		inputParameters.setProvince("*");
		inputParameters.setModuleName(module.getCode());
		
		if(delivery.getZone() != null && delivery.getZone().getCode()!=null) {
			inputParameters.setProvince(delivery.getZone().getCode());
		}
		
		if(distance!=null) {
			double ddistance = distance;
			long ldistance = (long)ddistance;
			inputParameters.setDistance(ldistance);
		}
		
		if(volume!=null) {
			inputParameters.setVolume((long)volume.doubleValue());
		}
		
		List<ShippingOption> options = quote.getShippingOptions();
		
		if(options == null) {
			options = new ArrayList<ShippingOption>();
			quote.setShippingOptions(options);
		}
		
		
		
		LOGGER.debug("Setting input parameters " + inputParameters.toString());
		
		
		KieSession kieSession=droolsBeanFactory.getKieSession(ResourceFactory.newClassPathResource("com/salesmanager/drools/rules/PriceByDistance.drl"));
		
		DecisionResponse resp = new DecisionResponse();
		
        kieSession.insert(inputParameters);
        kieSession.setGlobal("decision",resp);
        kieSession.fireAllRules();
        //System.out.println(resp.getCustomPrice());

		if(resp.getCustomPrice() != null) {

			ShippingOption shippingOption = new ShippingOption();
			
			
			shippingOption.setOptionPrice(new BigDecimal(resp.getCustomPrice()));
			shippingOption.setShippingModuleCode(MODULE_CODE);
			shippingOption.setOptionCode(MODULE_CODE);
			shippingOption.setOptionId(MODULE_CODE);

			options.add(shippingOption);
		}

		
		return options;
		
		
	}

/*	public StatelessKnowledgeSession getShippingPriceRule() {
		return shippingPriceRule;
	}

	public void setShippingPriceRule(StatelessKnowledgeSession shippingPriceRule) {
		this.shippingPriceRule = shippingPriceRule;
	}

	public KnowledgeBase getKbase() {
		return kbase;
	}

	public void setKbase(KnowledgeBase kbase) {
		this.kbase = kbase;
	}*/

}



```
