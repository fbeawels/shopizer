# ShippingDecisionPreProcessorImpl.java

## Review

## 1. Summary  
**Purpose** – `ShippingDecisionPreProcessorImpl` is a *pre‑processing* module for the shipping subsystem.  
It receives a shipping quote, a list of package details, and a host of contextual data (order total, delivery address, merchant store, configuration objects, etc.). Using a Drools rule‑based decision table it determines which integration module (e.g., UPS, FedEx, local carrier) should be used for the order, and sets that module on the `ShippingQuote`.

**Key Components**  
| Component | Role |
|-----------|------|
| `ShippingQuotePrePostProcessModule` | Interface that defines the contract for quote processors |
| `DroolsBeanFactory` | Factory that supplies a fresh `KieSession` for executing the rule set |
| `ShippingInputParameters` | DTO that holds the data needed by the Drools rule engine |
| `DecisionResponse` | Global Drools object that the rule updates with the chosen module name |
| `IntegrationModule` | Representation of a shipping carrier module; one of these will be selected |

**Notable Design Choices**  
* Rule‑based decision logic via Drools (`ShippingDecision.drl`).  
* Separation of *pre‑processing* from the actual quote calculation.  
* Use of Apache Commons `Validate` for defensive null‑checking.  

---

## 2. Detailed Description  
### Execution Flow  
1. **Validation** – The method asserts that all mandatory arguments (`delivery`, `currentModule`, `delivery.country`, `allModules`, `packages`) are non‑null/non‑empty.  
2. **Distance Extraction** – If a `quote` already contains a `DISTANCE_KEY` entry, the distance is retrieved.  
3. **Metrics Calculation** – Iterates over `PackageDetails` to compute:
   * **Weight** – sum of all package weights.  
   * **Volume** – the largest volume (length × width × height) among all packages.  
   * **Size** – the largest dimension (height, width, or length).  
4. **Input DTO Construction** – Fills a `ShippingInputParameters` object with the computed metrics and the destination country/province.  
5. **Rule Engine Invocation**  
   * A new `KieSession` is created for the rule file `ShippingDecision.drl`.  
   * `inputParameters` is inserted as a working memory fact.  
   * `DecisionResponse` is set as a global named `decision`.  
   * All rules are fired.  
   * The rule populates `DecisionResponse.moduleName`.  
6. **Module Selection** – The chosen module name is looked up in `allModules`. If found, `quote.setCurrentShippingModule(...)` is called.  
7. **Logging** – Debug logs are emitted throughout.

### Assumptions & Constraints  
* The rule file `ShippingDecision.drl` exists on the classpath under `com/salesmanager/drools/rules/`.  
* The `QuoteInformations` map (if present) holds a `DISTANCE_KEY` that maps to a `Double`.  
* Only the *largest* volume, weight, and size are considered; totals are not used.  
* Drools session is not explicitly disposed – the code relies on the container’s lifecycle.  

### Design Notes  
* **Stateless vs Stateful** – The implementation creates a fresh `KieSession` each time, treating the rule evaluation as stateless.  
* **Conversion to Long** – All numeric inputs are cast to `long` before insertion; precision loss is intentional but could be problematic for fractional values.  
* **Hard‑coded Rule Path** – The rule path is a string literal; any change to package layout would break the module.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `prePostProcessShippingQuotes(...)` | Main entry point; computes shipping metrics, runs Drools rules, selects the integration module. | *`ShippingQuote quote`* – the quote to be enriched. <br>*`List<PackageDetails> packages`* – items to ship. <br>*`BigDecimal orderTotal`* – total order value (unused). <br>*`Delivery delivery`* – destination address. <br>*`ShippingOrigin origin`* – shipping origin (unused). <br>*`MerchantStore store`* – merchant context (unused). <br>*`IntegrationConfiguration globalShippingConfiguration`* – global config (unused). <br>*`IntegrationModule currentModule`* – the module that is currently running. <br>*`ShippingConfiguration shippingConfiguration`* – shipping settings (unused). <br>*`List<IntegrationModule> allModules`* – all available shipping modules. <br>*`Locale locale`* – locale for i18n (unused). | None | Modifies `quote` by setting `currentShippingModule` if a module is selected. |
| `getModuleCode()` | Declares the module’s identifier. | None | `String` – `"shippingDecisionModule"` | None |

**Utility / Helper Logic** – The code performs in‑line calculations; no separate reusable helper methods are defined.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.kie.api.runtime.KieSession` | Third‑party (Drools) | Drools rule engine session. |
| `org.kie.internal.io.ResourceFactory` | Third‑party (Drools) | Loads rule files from the classpath. |
| `org.slf4j.Logger` / `LoggerFactory` | Third‑party (SLF4J) | Logging abstraction. |
| `org.apache.commons.lang3.StringUtils` | Third‑party (Apache Commons) | Utility for string checks. |
| `org.apache.commons.lang3.Validate` | Third‑party (Apache Commons) | Defensive null checking. |
| `com.salesmanager.core.business.configuration.DroolsBeanFactory` | Internal | Provides `KieSession`. |
| `com.salesmanager.core.model.*` | Internal | Domain models (`Delivery`, `PackageDetails`, etc.). |
| `com.salesmanager.core.modules.constants.Constants` | Internal | Holds `DISTANCE_KEY`. |
| `com.salesmanager.core.modules.integration.IntegrationException` | Internal | Declared for the processor interface. |
| `javax.inject.Inject` | Standard | CDI injection of `DroolsBeanFactory`. |

No OS‑specific or platform‑specific dependencies are present.

---

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns: the processor is solely responsible for determining the shipping module.  
* Defensive programming with `Validate` guards against null inputs.  
* Rule‑based flexibility: new carrier logic can be added by editing `ShippingDecision.drl` without code changes.  

### Potential Issues & Edge Cases  

1. **Null Map Handling** – `quote.getQuoteInformations()` is accessed without checking for `null`, which can throw a `NullPointerException`.  
2. **Drools Session Lifecycle** – The created `KieSession` is never disposed. Repeated invocations may leak memory or exhaust resources.  
3. **Metric Calculation** – Only the *maximum* volume and size are used; total weight is summed but truncated to `long`. This may not reflect real shipping calculations (most carriers use total volume/weight).  
4. **Hard‑coded Rule Path** – `"com/salesmanager/drools/rules/ShippingDecision.drl"` may be misspelled (`salesmanager` vs `salesmanager`) and is brittle to package changes.  
5. **Unused Parameters** – Many arguments (`orderTotal`, `origin`, `store`, `globalShippingConfiguration`, `shippingConfiguration`, `locale`) are unused, suggesting either incomplete implementation or a mismatch between the interface contract and actual usage.  
6. **System.out.println** – Debug output should be routed through SLF4J only.  
7. **Exception Handling** – No `try/catch` around Drools execution; any rule error will propagate as a runtime exception.  
8. **Global Variable Management** – `DecisionResponse` is used as a global; if multiple threads use the same session concurrently this could lead to race conditions.  

### Suggested Enhancements  

| Area | Recommendation |
|------|----------------|
| **Session Management** | Call `kieSession.dispose()` after `fireAllRules()`. |
| **Null Safety** | Add a null‑check for `quote.getQuoteInformations()` before accessing it. |
| **Metrics** | Clarify business rules: use total volume/weight or keep the largest? Provide a config flag. |
| **Rule Path** | Externalize the DRL path to a property or use class‑path resource resolution via classloader. |
| **Logging** | Remove `System.out.println` and ensure all logs use the logger. |
| **Parameter Utilization** | Either remove unused parameters from the interface or incorporate them into the decision logic. |
| **Thread Safety** | Use a *stateless* `KieSession` or guard the global variable with thread‑local storage. |
| **Unit Tests** | Add tests that exercise various combinations of package sizes, weights, distances, and verify correct module selection. |
| **Exception Handling** | Wrap Drools invocation in a try/catch and translate runtime issues into `IntegrationException` where appropriate. |

### Summary  
`ShippingDecisionPreProcessorImpl` delivers a clean, rule‑driven approach to selecting a shipping carrier. While the core logic is straightforward, the implementation could benefit from tighter null handling, proper session disposal, and clearer documentation of metric calculations. Addressing these areas will improve robustness, maintainability, and future extensibility.

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
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.modules.constants.Constants;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuotePrePostProcessModule;

/**
 * Decides which shipping method is going to be used based on a decision table
 * @author carlsamson
 *
 */
public class ShippingDecisionPreProcessorImpl implements ShippingQuotePrePostProcessModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ShippingDecisionPreProcessorImpl.class);
	
	private final static String MODULE_CODE = "shippingDecisionModule";
	
	@Inject
	private DroolsBeanFactory droolsBeanFactory;
	
	//private StatelessKnowledgeSession shippingMethodDecision;
	
	//private KnowledgeBase kbase;
	
	//@Inject
	//KieContainer kieShippingDecisionContainer;
	
	@Override
	public void prePostProcessShippingQuotes(
			ShippingQuote quote,
			List<PackageDetails> packages, 
			BigDecimal orderTotal,
			Delivery delivery, 
			ShippingOrigin origin, 
			MerchantStore store,
			IntegrationConfiguration globalShippingConfiguration,
			IntegrationModule currentModule,
			ShippingConfiguration shippingConfiguration,
			List<IntegrationModule> allModules, 
			Locale locale)
			throws IntegrationException {
		
		
		Validate.notNull(delivery, "Delivery cannot be null");
		Validate.notNull(currentModule, "IntegrationModule cannot be null");
		Validate.notNull(delivery.getCountry(), "Delivery.country cannot be null");
		Validate.notNull(allModules, "List<IntegrationModule> cannot be null");
		Validate.notNull(packages, "packages cannot be null");
		Validate.notEmpty(packages, "packages cannot be empty");
		
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
		//calculate weight, volume and largest size
		for(PackageDetails pack : packages) {
			weight = weight + pack.getShippingWeight();
			Double tmpVolume = pack.getShippingHeight() * pack.getShippingLength() * pack.getShippingWidth();
			if(volume == null || tmpVolume > volume) { //take the largest volume
				volume = tmpVolume;
			} 
			//largest size
			List<Double> sizeList = new ArrayList<Double>();
			sizeList.add(pack.getShippingHeight());
			sizeList.add(pack.getShippingLength());
			sizeList.add(pack.getShippingWidth());
			Double maxSize = Collections.max(sizeList);
			if(size==null || maxSize > size) {
				size = maxSize;
			}
		}
		
		//Build a ShippingInputParameters
		ShippingInputParameters inputParameters = new ShippingInputParameters();
		
		inputParameters.setWeight((long)weight.doubleValue());
		inputParameters.setCountry(delivery.getCountry().getIsoCode());
		if(delivery.getZone()!=null && delivery.getZone().getCode()!=null) {
			inputParameters.setProvince(delivery.getZone().getCode());
		} else {
			inputParameters.setProvince(delivery.getState());
		}
		//inputParameters.setModuleName(currentModule.getCode());
		
		
		if(size!=null) {
			inputParameters.setSize((long)size.doubleValue());
		}
		
		if(distance!=null) {
			double ddistance = distance;
			long ldistance = (long)ddistance;
			inputParameters.setDistance(ldistance);
		}
		
		if(volume!=null) {
			inputParameters.setVolume((long)volume.doubleValue());
		}
		
		LOGGER.debug("Setting input parameters " + inputParameters.toString());
		System.out.println(inputParameters.toString());
		
		
		/**
		 * New code
		 */
		
		KieSession kieSession=droolsBeanFactory.getKieSession(ResourceFactory.newClassPathResource("com/salesmanager/drools/rules/ShippingDecision.drl"));
		
		DecisionResponse resp = new DecisionResponse();
		
        kieSession.insert(inputParameters);
        kieSession.setGlobal("decision",resp);
        kieSession.fireAllRules();
        //System.out.println(resp.getModuleName());
        inputParameters.setModuleName(resp.getModuleName());

		LOGGER.debug("Using shipping nodule " + inputParameters.getModuleName());
		
		if(!StringUtils.isBlank(inputParameters.getModuleName())) {
			for(IntegrationModule toBeUsed : allModules) {
				if(toBeUsed.getCode().equals(inputParameters.getModuleName())) {
					quote.setCurrentShippingModule(toBeUsed);
					break;
				}
			}
		}
		
	}


	@Override
	public String getModuleCode() {
		return MODULE_CODE;
	}
	







}



```
