# ShippingServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`ShippingServiceImpl` implements the `ShippingService` interface and is the central orchestration point for all shipping‑related logic in the SalesManager core module. It:

- Loads and persists store‑level shipping configuration (`ShippingConfiguration`).
- Manages shipping module configurations (`IntegrationConfiguration`) and module registrations.
- Calculates shipping quotes for a given cart, delivery address, and set of products.
- Exposes helper methods for country support, shipping package details, and metadata retrieval.

**Key Components**

| Component | Role |
|-----------|------|
| `ShippingConfiguration` | Global store‑wide settings (free shipping, tax on shipping, package type, etc.). |
| `IntegrationConfiguration` | Module‑specific parameters for a particular shipping provider. |
| `ShippingQuoteModule` | Interface for concrete shipping provider implementations. |
| `ShippingQuotePrePostProcessModule` | Hooks that run before/after quote calculation (e.g., distance calculation, module selection). |
| `ShippingOriginService` | Provides the shipping origin (store address). |
| `ShippingQuoteService` | Persistence of individual shipping quote options. |
| `Packaging` | Utility that converts products into `PackageDetails` (either by item or by box). |
| `CountryService`, `LanguageService`, `PricingService` | Domain services used for lookups, localization, and price formatting. |
| `MerchantConfigurationService`, `ModuleConfigurationService`, `Encryption` | Persist and encrypt module configurations. |

The service uses **Spring** (`@Service`, `@Inject`, `@Resource`) and **Jackson** for JSON handling. It also relies on **Apache Commons** (`Validate`, `StringUtils`, `CollectionUtils`) and **JSON.simple** for legacy JSON parsing.

## 2. Detailed Description  

### Flow of Execution

1. **Initialization** – All dependencies are injected by Spring.  
2. **Quote Calculation** – `getShippingQuote(...)` orchestrates the entire quote process:
   - Validates input (store, delivery, products, language).
   - Loads global shipping config and module configs.
   - Determines shipping origin and type (national/international).
   - Applies pre‑processors to allow modules to adjust the request.
   - Calls the chosen `ShippingQuoteModule` to obtain a list of `ShippingOption`s.
   - Filters options according to the chosen `ShippingOptionPriceType` (HIGHEST, LEAST, ALL).
   - Persists each `ShippingOption` as a `Quote` record.
   - Applies post‑processors (e.g., final adjustments, analytics).
3. **Configuration Persistence** – `saveShippingConfiguration` and `saveCustomShippingConfiguration` serialize the objects to JSON and store them encrypted.
4. **Metadata** – `getShippingMetaData` aggregates data needed by the front‑end: supported countries, modules, pre/post processors, and distance‑module flag.

### Assumptions & Constraints

- **Single Active Module**: The service picks the first active module from the configuration map. If a pre‑processor changes the module, the module instance is updated.
- **JSON Format**: Global and module configurations are stored as JSON strings; any format error throws a `ServiceException`.
- **Encryption**: All module configurations are encrypted using `Encryption`. Decryption is required before they can be used.
- **Country List**: Supported countries are stored as a JSON array of ISO codes. If missing, no restrictions are applied (except for national mode).
- **Null Safety**: The code performs extensive null checks, but some paths (e.g., `shippingConfiguration == null`) result in silent defaults.
- **Concurrency**: The service is a singleton Spring bean; thread safety depends on underlying services. No explicit synchronization is used.

### Architecture & Design Choices

- **Service‑Oriented**: The logic is split into domain services (`ShippingOriginService`, `ShippingQuoteService`, etc.) that keep this class thin.
- **Plugin Architecture**: Shipping modules are discovered via Spring beans (`@Resource(name="shippingModules")`). Pre/post processors can be added without changing the core logic.
- **Data‑Driven Configuration**: Store‑wide settings are stored in a generic `MerchantConfiguration` table, enabling multi‑tenant isolation.
- **Error Handling**: Most methods wrap low‑level exceptions into `ServiceException`, but some error handling (e.g., during pre‑processor module switching) is minimal and could lead to silent failures.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getShippingConfiguration(MerchantStore)` | Retrieves global shipping config for a store. | `store` | `ShippingConfiguration` | None |
| `getShippingConfiguration(String, MerchantStore)` | Retrieves config of a specific shipping module. | `moduleCode`, `store` | `IntegrationConfiguration` | None |
| `getCustomShippingConfiguration(MerchantStore)` | Retrieves encrypted custom config. | `moduleCode`, `store` | `Custom config object` | None |
| `saveShippingConfiguration(ShippingConfiguration, MerchantStore)` | Persists global config. | `config`, `store` | None | Updates `MerchantConfiguration` row |
| `saveCustomShippingConfiguration(String, IntegrationConfiguration, MerchantStore)` | Persists module‑specific config. | `moduleCode`, `config`, `store` | None | Updates `MerchantConfiguration` row (encrypted) |
| `getSupportedCountries(MerchantStore)` | Reads list of ISO country codes supported for shipping. | `store` | `List<String>` | None |
| `setSupportedCountries(MerchantStore, List<String>)` | Persists supported country list. | `store`, `countryCodes` | None | Updates `MerchantConfiguration` |
| `requiresShipping(List<ShoppingCartItem>, MerchantStore)` | Checks if any item needs shipping. | `items`, `store` | `boolean` | None |
| `getSupportedCountries` / `getShipToCountryList` | Compute country lists based on config. | `store`, `language` | `List<Country>` | None |
| `getSupportedCountries` (overloaded) | Helper that converts list of codes to JSON. | `store`, `codes` | None | Persists config |
| `calculateOrderTotal(List<ShippingProduct>, MerchantStore)` | Computes cart total (including quantity). | `products`, `store` | `BigDecimal` | None |
| `getPackagesDetails(List<ShippingProduct>, MerchantStore)` | Generates `PackageDetails` using `Packaging`. | `products`, `store` | `List<PackageDetails>` | None |
| `getShipToCountryList(MerchantStore, Language)` | Returns list of countries a store can ship to. | `store`, `lang` | `List<Country>` | None |
| `hasTaxOnShipping(MerchantStore)` | Returns whether tax applies to shipping. | `store` | `boolean` | None |
| `getShippingMetaData(MerchantStore)` | Aggregates all metadata needed by UI. | `store` | `ShippingMetaData` | None |
| `hasTaxOnShipping(MerchantStore)` | Convenience wrapper for tax flag. | `store` | `boolean` | None |

### Notable Method Implementations

- **`getShippingQuote`** – The most complex method. Handles validation, pre/post processors, module switching, option filtering, price formatting, and persistence.  
- **`getPackagesDetails`** – Delegates to `packaging` to build package info based on `ShippingPackageType`.  
- **`requiresShipping`** – Quick check on whether the cart contains any physical shippable product.  
- **`getShippingMetaData`** – Uses Java 8 `Optional` to avoid `null`‑based logic.

## 4. Dependencies  

| Library | Package | Usage |
|---------|---------|-------|
| Spring Framework | `org.springframework` | Bean injection, `@Service`, `@Inject`, `@Resource`. |
| Jackson | `com.fasterxml.jackson.databind.ObjectMapper` | JSON (de)serialization for config objects. |
| JSON.simple | `org.json.simple.parser.JSONValue`, `JSONArray` | Legacy parsing of supported‑country list. |
| Apache Commons | `org.apache.commons.lang3.*`, `org.apache.commons.collections4.CollectionUtils` | String utilities, collection helpers, input validation. |
| Custom Services | `MerchantConfigurationService`, `ModuleConfigurationService`, `Encryption`, `ShippingOriginService`, `ShippingQuoteService`, `PricingService`, `CountryService`, `LanguageService` | Domain logic, persistence, encryption, localization. |
| Logging | `org.slf4j.Logger` | Error logging. |

The service mixes **Jackson** and **JSON.simple**. The former is modern, while the latter is legacy and may lead to confusion or errors if input format changes.

## 4. Additional Notes & Recommendations  

### Strengths  
- **Extensibility** – The plugin architecture makes it straightforward to add new shipping providers or processors.  
- **Domain‑Separation** – Core calculations are delegated to smaller services, keeping the class cohesive.  
- **Configuration Security** – Encrypting module configs is a good practice for multi‑tenant data.

### Areas for Improvement  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded “first active module” logic** | Only one module can be active at a time; switching may be error‑prone. | Expose a deterministic module selection strategy (e.g., priority ordering) and unit‑test module switching. |
| **Exception handling in pre/post processors** | Some errors are swallowed or logged only, leading to potentially incomplete quotes. | Convert all critical errors to `ServiceException` and return a clear error status (`ShippingQuote.ERROR`). |
| **JSON.simple Usage** | Mixing JSON libraries increases maintenance cost and may produce type mismatches. | Replace legacy `JSON.simple` parsing with Jackson (`ObjectMapper.readTree`) for consistency. |
| **Null Defaulting** | Silently falling back to defaults can mask misconfiguration (e.g., missing `ShippingConfiguration`). | Explicitly check and throw `ServiceException` if required config is absent. |
| **Distance Module Flag** | The flag is derived from a pre‑processor code; if the module list changes, the flag may become stale. | Compute `useDistanceModule` directly from the processor list each time. |
| **Thread Safety** | Singleton bean accessing mutable state (`shippingModulePreProcessors`, `shippingModulePostProcessors`) – though they are injected once, no defensive copying. | Document that these collections are read‑only or make defensive copies. |
| **Logging** – The quote creation section logs but does not persist errors. | Users may not be aware of calculation failures. | Persist error logs (e.g., via `MerchantLogService`) or surface error messages in the quote object. |
| **IP Address Retrieval** | Uses `UserContext` which may be null. | Provide a fallback (e.g., empty string) and document the expected context. |
| **Date Handling** | Several commented‑out conversions (`DateUtil.formatDate`) and missing date parsing. | Remove dead code or implement proper date conversions. |
| **Price Conversion** | `option.getOptionPriceText` uses `pricingService.getDisplayAmount`, but formatting could vary per currency. | Ensure locale‑aware formatting by passing `Locale` derived from language/store. |
| **Optional/Stream API** | Only used in `getShippingMetaData`. | Extend its usage to other parts for consistency. |

### Testability

The heavy reliance on external services makes unit testing of `ShippingServiceImpl` cumbersome. Suggested strategies:

- **Mock all injected services** using Mockito or a Spring test context.
- **Wrap the pre/post processor invocation** in separate methods so they can be unit‑tested independently.
- **Parameterize the module selection logic** (e.g., expose a `ShippingModuleSelector` bean) to isolate the selection algorithm.

### Documentation

The source code has no JavaDoc, making it hard for developers unfamiliar with the domain to understand each method’s contract. Adding class‑level and method‑level Javadoc would greatly improve maintainability.

---

**Overall Assessment**  
`ShippingServiceImpl` is a feature‑rich, well‑structured orchestration class that leverages Spring and a plugin architecture. Its biggest strengths are extensibility and domain separation. However, the mixing of legacy JSON libraries, sparse error handling in processor switching, and the simplistic module selection logic are the main pain points that should be addressed to increase robustness and clarity.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.shipping;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Optional;

import javax.annotation.Resource;
import javax.inject.Inject;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.json.simple.JSONArray;
import org.json.simple.JSONValue;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.salesmanager.core.business.constants.ShippingConstants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.business.services.reference.loader.ConfigurationModulesLoader;
import com.salesmanager.core.business.services.system.MerchantConfigurationService;
import com.salesmanager.core.business.services.system.ModuleConfigurationService;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.common.UserContext;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.Quote;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingMetaData;
import com.salesmanager.core.model.shipping.ShippingOption;
import com.salesmanager.core.model.shipping.ShippingOptionPriceType;
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingPackageType;
import com.salesmanager.core.model.shipping.ShippingProduct;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.shipping.ShippingSummary;
import com.salesmanager.core.model.shipping.ShippingType;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.model.system.CustomIntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.model.system.MerchantConfiguration;
import com.salesmanager.core.modules.integration.IntegrationException;
import com.salesmanager.core.modules.integration.shipping.model.Packaging;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuoteModule;
import com.salesmanager.core.modules.integration.shipping.model.ShippingQuotePrePostProcessModule;
import com.salesmanager.core.modules.utils.Encryption;


@Service("shippingService")
public class ShippingServiceImpl implements ShippingService {

	private static final Logger LOGGER = LoggerFactory.getLogger(ShippingServiceImpl.class);
	
	
	private final static String SUPPORTED_COUNTRIES = "SUPPORTED_CNTR";
	private final static String SHIPPING_MODULES = "SHIPPING";
	private final static String SHIPPING_DISTANCE = "shippingDistanceModule";

	
	@Inject
	private MerchantConfigurationService merchantConfigurationService;
	

	@Inject
	private PricingService pricingService;
	
	@Inject
	private ModuleConfigurationService moduleConfigurationService;
	
	@Inject
	private Packaging packaging;
	
	@Inject
	private CountryService countryService;
	
	@Inject
	private LanguageService languageService;
	
	@Inject
	private Encryption encryption;

	@Inject
	private ShippingOriginService shippingOriginService;
	
	@Inject
	private ShippingQuoteService shippingQuoteService;
	
	@Inject
	@Resource(name="shippingModules")
	private Map<String,ShippingQuoteModule> shippingModules;
	
	//shipping pre-processors
	@Inject
	@Resource(name="shippingModulePreProcessors")
	private List<ShippingQuotePrePostProcessModule> shippingModulePreProcessors;
	
	//shipping post-processors
	@Inject
	@Resource(name="shippingModulePostProcessors")
	private List<ShippingQuotePrePostProcessModule> shippingModulePostProcessors;
	
	@Override
	public ShippingConfiguration getShippingConfiguration(MerchantStore store) throws ServiceException {

		MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(ShippingConstants.SHIPPING_CONFIGURATION, store);
		
		ShippingConfiguration shippingConfiguration = null;
		
		if(configuration!=null) {
			String value = configuration.getValue();
			
			ObjectMapper mapper = new ObjectMapper();
			try {
				shippingConfiguration = mapper.readValue(value, ShippingConfiguration.class);
			} catch(Exception e) {
				throw new ServiceException("Cannot parse json string " + value);
			}
		}
		return shippingConfiguration;
		
	}
	
	@Override
	public IntegrationConfiguration getShippingConfiguration(String moduleCode, MerchantStore store) throws ServiceException {

		
		Map<String,IntegrationConfiguration> configuredModules = getShippingModulesConfigured(store);
		if(configuredModules!=null) {
			for(String key : configuredModules.keySet()) {
				if(key.equals(moduleCode)) {
					return configuredModules.get(key);	
				}
			}
		}
		
		return null;
		
	}
	
	@Override
	public CustomIntegrationConfiguration getCustomShippingConfiguration(String moduleCode, MerchantStore store) throws ServiceException {

		
		ShippingQuoteModule quoteModule = shippingModules.get(moduleCode);
		if(quoteModule==null) {
			return null;
		}
		return quoteModule.getCustomModuleConfiguration(store);
		
	}
	
	@Override
	public void saveShippingConfiguration(ShippingConfiguration shippingConfiguration, MerchantStore store) throws ServiceException {
		
		MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(ShippingConstants.SHIPPING_CONFIGURATION, store);

		if(configuration==null) {
			configuration = new MerchantConfiguration();
			configuration.setMerchantStore(store);
			configuration.setKey(ShippingConstants.SHIPPING_CONFIGURATION);
		}
		
		String value = shippingConfiguration.toJSONString();
		configuration.setValue(value);
		merchantConfigurationService.saveOrUpdate(configuration);
		
	}
	
	@Override
	public void saveCustomShippingConfiguration(String moduleCode, CustomIntegrationConfiguration shippingConfiguration, MerchantStore store) throws ServiceException {
		
		
		ShippingQuoteModule quoteModule = shippingModules.get(moduleCode);
		if(quoteModule==null) {
			throw new ServiceException("Shipping module " + moduleCode + " does not exist");
		}
		
		String configurationValue = shippingConfiguration.toJSONString();
		
		
		try {

			MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(moduleCode, store);
	
			if(configuration==null) {

				configuration = new MerchantConfiguration();
				configuration.setKey(moduleCode);
				configuration.setMerchantStore(store);
			}
			configuration.setValue(configurationValue);
			merchantConfigurationService.saveOrUpdate(configuration);
		
		} catch (Exception e) {
			throw new IntegrationException(e);
		}

		
		
	}
	

	@Override
	public List<IntegrationModule> getShippingMethods(MerchantStore store) throws ServiceException {
		
		List<IntegrationModule> modules =  moduleConfigurationService.getIntegrationModules(SHIPPING_MODULES);
		List<IntegrationModule> returnModules = new ArrayList<IntegrationModule>();
		
		for(IntegrationModule module : modules) {
			if(module.getRegionsSet().contains(store.getCountry().getIsoCode())
					|| module.getRegionsSet().contains("*")) {
				
				returnModules.add(module);
			}
		}
		
		return returnModules;
	}
	
	@Override
	public void saveShippingQuoteModuleConfiguration(IntegrationConfiguration configuration, MerchantStore store) throws ServiceException {
		
			//validate entries
			try {
				
				String moduleCode = configuration.getModuleCode();
				ShippingQuoteModule quoteModule = (ShippingQuoteModule)shippingModules.get(moduleCode);
				if(quoteModule==null) {
					throw new ServiceException("Shipping quote module " + moduleCode + " does not exist");
				}
				quoteModule.validateModuleConfiguration(configuration, store);
				
			} catch (IntegrationException ie) {
				throw ie;
			}
			
			try {
				Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
				MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(SHIPPING_MODULES, store);
				if(merchantConfiguration!=null) {
					if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
						
						String decrypted = encryption.decrypt(merchantConfiguration.getValue());
						modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
					}
				} else {
					merchantConfiguration = new MerchantConfiguration();
					merchantConfiguration.setMerchantStore(store);
					merchantConfiguration.setKey(SHIPPING_MODULES);
				}
				modules.put(configuration.getModuleCode(), configuration);
				
				String configs =  ConfigurationModulesLoader.toJSONString(modules);
				
				String encrypted = encryption.encrypt(configs);
				merchantConfiguration.setValue(encrypted);
				merchantConfigurationService.saveOrUpdate(merchantConfiguration);
				
			} catch (Exception e) {
				throw new ServiceException(e);
			}
	}
	
	
	@Override
	public void removeShippingQuoteModuleConfiguration(String moduleCode, MerchantStore store) throws ServiceException {
		
		

		try {
			Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(SHIPPING_MODULES, store);
			if(merchantConfiguration!=null) {
				if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
					String decrypted = encryption.decrypt(merchantConfiguration.getValue());
					modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
				}
				
				modules.remove(moduleCode);
				String configs =  ConfigurationModulesLoader.toJSONString(modules);
				String encrypted = encryption.encrypt(configs);
				merchantConfiguration.setValue(encrypted);
				merchantConfigurationService.saveOrUpdate(merchantConfiguration);
				
				
			} 
			
			MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(moduleCode, store);
			
			if(configuration!=null) {//custom module

				merchantConfigurationService.delete(configuration);
			}

			
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	
	}
	
	@Override
	public void removeCustomShippingQuoteModuleConfiguration(String moduleCode, MerchantStore store) throws ServiceException {
		
		

		try {
			
			removeShippingQuoteModuleConfiguration(moduleCode,store);
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(moduleCode, store);
			if(merchantConfiguration!=null) {
				merchantConfigurationService.delete(merchantConfiguration);
			} 
			
			
		} catch (Exception e) {
			throw new ServiceException(e);
		}
	
	}
	
	@Override
	public Map<String,IntegrationConfiguration> getShippingModulesConfigured(MerchantStore store) throws ServiceException {
		try {
			

			Map<String,IntegrationConfiguration> modules = new HashMap<String,IntegrationConfiguration>();
			MerchantConfiguration merchantConfiguration = merchantConfigurationService.getMerchantConfiguration(SHIPPING_MODULES, store);
			if(merchantConfiguration!=null) {
				if(!StringUtils.isBlank(merchantConfiguration.getValue())) {
					String decrypted = encryption.decrypt(merchantConfiguration.getValue());
					modules = ConfigurationModulesLoader.loadIntegrationConfigurations(decrypted);
					
				}
			}
			return modules;
		
		
		} catch (Exception e) {
			throw new ServiceException(e);
		}
		
	}
	
	@Override
	public ShippingSummary getShippingSummary(MerchantStore store, ShippingQuote shippingQuote, ShippingOption selectedShippingOption) throws ServiceException {
		
		ShippingSummary shippingSummary = new ShippingSummary();
		shippingSummary.setFreeShipping(shippingQuote.isFreeShipping());
		shippingSummary.setHandling(shippingQuote.getHandlingFees());
		shippingSummary.setShipping(selectedShippingOption.getOptionPrice());
		shippingSummary.setShippingModule(shippingQuote.getShippingModuleCode());
		shippingSummary.setShippingOption(selectedShippingOption.getDescription());
		
		return shippingSummary;
	}

	@Override
	public ShippingQuote getShippingQuote(Long shoppingCartId, MerchantStore store, Delivery delivery, List<ShippingProduct> products, Language language) throws ServiceException  {
		
		
		//ShippingConfiguration -> Global configuration of a given store
		//IntegrationConfiguration -> Configuration of a given module
		//IntegrationModule -> The concrete module as defined in integrationmodules.properties
		
		//delivery without postal code is accepted
		Validate.notNull(store,"MerchantStore must not be null");
		Validate.notNull(delivery,"Delivery must not be null");
		Validate.notEmpty(products,"products must not be empty");
		Validate.notNull(language,"Language must not be null");
		
		
		
		ShippingQuote shippingQuote = new ShippingQuote();
		ShippingQuoteModule shippingQuoteModule = null;
		
		try {
			
			
			if(StringUtils.isBlank(delivery.getPostalCode())) {
				shippingQuote.getWarnings().add("No postal code in delivery address");
				shippingQuote.setShippingReturnCode(ShippingQuote.NO_POSTAL_CODE);
			}
		
			//get configuration
			ShippingConfiguration shippingConfiguration = getShippingConfiguration(store);
			ShippingType shippingType = ShippingType.INTERNATIONAL;
			
			/** get shipping origin **/
			ShippingOrigin shippingOrigin = shippingOriginService.getByStore(store);
			if(shippingOrigin == null || !shippingOrigin.isActive()) {
				shippingOrigin = new ShippingOrigin();
				shippingOrigin.setAddress(store.getStoreaddress());
				shippingOrigin.setCity(store.getStorecity());
				shippingOrigin.setCountry(store.getCountry());
				shippingOrigin.setPostalCode(store.getStorepostalcode());
				shippingOrigin.setState(store.getStorestateprovince());
				shippingOrigin.setZone(store.getZone());
			}
			
			
			if(shippingConfiguration==null) {
				shippingConfiguration = new ShippingConfiguration();
			}
			
			if(shippingConfiguration.getShippingType()!=null) {
					shippingType = shippingConfiguration.getShippingType();
			}

			//look if customer country code excluded
			Country shipCountry = delivery.getCountry();
			
			//a ship to country is required
			Validate.notNull(shipCountry,"Ship to Country cannot be null");
			Validate.notNull(store.getCountry(), "Store Country canot be null");
			
			if(shippingType.name().equals(ShippingType.NATIONAL.name())){
				//customer country must match store country
				if(!shipCountry.getIsoCode().equals(store.getCountry().getIsoCode())) {
					shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_TO_SELECTED_COUNTRY + " " + shipCountry.getIsoCode());
					return shippingQuote;
				}
			} else if(shippingType.name().equals(ShippingType.INTERNATIONAL.name())){
				
				//customer shipping country code must be in accepted list
				List<String> supportedCountries = this.getSupportedCountries(store);
				if(!supportedCountries.contains(shipCountry.getIsoCode())) {
					shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_TO_SELECTED_COUNTRY + " " + shipCountry.getIsoCode());
					return shippingQuote;
				}
			}
			
			//must have a shipping module configured
			Map<String, IntegrationConfiguration> modules = this.getShippingModulesConfigured(store);
			if(modules == null){
				shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_MODULE_CONFIGURED);
				return shippingQuote;
			}

			
			/** uses this module name **/
			String moduleName = null;
			IntegrationConfiguration configuration = null;
			for(String module : modules.keySet()) {
				moduleName = module;
				configuration = modules.get(module);
				//use the first active module
				if(configuration.isActive()) {
					shippingQuoteModule = shippingModules.get(module);
					if(shippingQuoteModule instanceof ShippingQuotePrePostProcessModule) {
						shippingQuoteModule = null;
						continue;
					} else {
						break;
					}
				}
			}
			
			if(shippingQuoteModule==null){
				shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_MODULE_CONFIGURED);
				return shippingQuote;
			}
			
			/** merchant module configs **/
			List<IntegrationModule> shippingMethods = this.getShippingMethods(store);
			IntegrationModule shippingModule = null;
			for(IntegrationModule mod : shippingMethods) {
				if(mod.getCode().equals(moduleName)){
					shippingModule = mod;
					break;
				}
			}
			
			/** general module configs **/
			if(shippingModule==null) {
				shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_MODULE_CONFIGURED);
				return shippingQuote;
			}
			
			//calculate order total
			BigDecimal orderTotal = calculateOrderTotal(products,store);
			List<PackageDetails> packages = getPackagesDetails(products, store);
			
			//free shipping ?
			boolean freeShipping = false;
			if(shippingConfiguration.isFreeShippingEnabled()) {
				BigDecimal freeShippingAmount = shippingConfiguration.getOrderTotalFreeShipping();
				if(freeShippingAmount!=null) {
					if(orderTotal.doubleValue()>freeShippingAmount.doubleValue()) {
						if(shippingConfiguration.getFreeShippingType() == ShippingType.NATIONAL) {
							if(store.getCountry().getIsoCode().equals(shipCountry.getIsoCode())) {
								freeShipping = true;
								shippingQuote.setFreeShipping(true);
								shippingQuote.setFreeShippingAmount(freeShippingAmount);
								return shippingQuote;
							}
						} else {//international all
							freeShipping = true;
							shippingQuote.setFreeShipping(true);
							shippingQuote.setFreeShippingAmount(freeShippingAmount);
							return shippingQuote;
						}
	
					}
				}
			}
			

			//handling fees
			BigDecimal handlingFees = shippingConfiguration.getHandlingFees();
			if(handlingFees!=null) {
				shippingQuote.setHandlingFees(handlingFees);
			}
			
			//tax basis
			shippingQuote.setApplyTaxOnShipping(shippingConfiguration.isTaxOnShipping());
			

			Locale locale = languageService.toLocale(language, store);
			
			//invoke pre processors
			//the main pre-processor determines at runtime the shipping module
			//also available distance calculation
			if(!CollectionUtils.isEmpty(shippingModulePreProcessors)) {
				for(ShippingQuotePrePostProcessModule preProcessor : shippingModulePreProcessors) {
					//System.out.println("Using pre-processor " + preProcessor.getModuleCode());
					preProcessor.prePostProcessShippingQuotes(shippingQuote, packages, orderTotal, delivery, shippingOrigin, store, configuration, shippingModule, shippingConfiguration, shippingMethods, locale);
					//TODO switch module if required
					if(shippingQuote.getCurrentShippingModule()!=null && !shippingQuote.getCurrentShippingModule().getCode().equals(shippingModule.getCode())) {
						shippingModule = shippingQuote.getCurrentShippingModule();//determines the shipping module
						configuration = modules.get(shippingModule.getCode());
						if(configuration!=null) {
							if(configuration.isActive()) {
								moduleName = shippingModule.getCode();
								shippingQuoteModule = this.shippingModules.get(shippingModule.getCode());
								configuration = modules.get(shippingModule.getCode());
							} //TODO use default
						}
						
					}
				}
			}

			//invoke module
			List<ShippingOption> shippingOptions = null;
					
			try {
				shippingOptions = shippingQuoteModule.getShippingQuotes(shippingQuote, packages, orderTotal, delivery, shippingOrigin, store, configuration, shippingModule, shippingConfiguration, locale);
			} catch(Exception e) {
				LOGGER.error("Error while calculating shipping : " + e.getMessage(), e);
/*				merchantLogService.save(
						new MerchantLog(store,
								"Can't process " + shippingModule.getModule()
								+ " -> "
								+ e.getMessage()));
				shippingQuote.setQuoteError(e.getMessage());
				shippingQuote.setShippingReturnCode(ShippingQuote.ERROR);
				return shippingQuote;*/
			}
			
			if(shippingOptions==null && !StringUtils.isBlank(delivery.getPostalCode())) {
				
				//absolutely need to use in this case store pickup or other default shipping quote
				shippingQuote.setShippingReturnCode(ShippingQuote.NO_SHIPPING_TO_SELECTED_COUNTRY);
			}
			
			
			shippingQuote.setShippingModuleCode(moduleName);	
			
			//filter shipping options
			ShippingOptionPriceType shippingOptionPriceType = shippingConfiguration.getShippingOptionPriceType();
			ShippingOption selectedOption = null;
			
			if(shippingOptions!=null) {
				
				for(ShippingOption option : shippingOptions) {
					if(selectedOption==null) {
						selectedOption = option;
					}
					//set price text
					String priceText = pricingService.getDisplayAmount(option.getOptionPrice(), store);
					option.setOptionPriceText(priceText);
					option.setShippingModuleCode(moduleName);
				
					if(StringUtils.isBlank(option.getOptionName())) {
						
						String countryName = delivery.getCountry().getName();
						if(countryName == null) {
							Map<String,Country> deliveryCountries = countryService.getCountriesMap(language);
							Country dCountry = deliveryCountries.get(delivery.getCountry().getIsoCode());
							if(dCountry!=null) {
								countryName = dCountry.getName();
							} else {
								countryName = delivery.getCountry().getIsoCode();
							}
						}
							option.setOptionName(countryName);		
					}
				
					if(shippingOptionPriceType.name().equals(ShippingOptionPriceType.HIGHEST.name())) {

						if (option.getOptionPrice()
								.longValue() > selectedOption
								.getOptionPrice()
								.longValue()) {
							selectedOption = option;
						}
					}

				
					if(shippingOptionPriceType.name().equals(ShippingOptionPriceType.LEAST.name())) {

						if (option.getOptionPrice()
								.longValue() < selectedOption
								.getOptionPrice()
								.longValue()) {
							selectedOption = option;
						}
					}
					
				
					if(shippingOptionPriceType.name().equals(ShippingOptionPriceType.ALL.name())) {
	
						if (option.getOptionPrice()
								.longValue() < selectedOption
								.getOptionPrice()
								.longValue()) {
							selectedOption = option;
						}
					}

				}
				
				shippingQuote.setSelectedShippingOption(selectedOption);
				
				if(selectedOption!=null && !shippingOptionPriceType.name().equals(ShippingOptionPriceType.ALL.name())) {
					shippingOptions = new ArrayList<ShippingOption>();
					shippingOptions.add(selectedOption);
				}

			}
			
			/** set final delivery address **/
			shippingQuote.setDeliveryAddress(delivery);
			
			shippingQuote.setShippingOptions(shippingOptions);
			
			/** post processors **/
			//invoke pre processors
			if(!CollectionUtils.isEmpty(shippingModulePostProcessors)) {
				for(ShippingQuotePrePostProcessModule postProcessor : shippingModulePostProcessors) {
					//get module info
					
					//get module configuration
					IntegrationConfiguration integrationConfiguration = modules.get(postProcessor.getModuleCode());
					
					IntegrationModule postProcessModule = null;
					for(IntegrationModule mod : shippingMethods) {
						if(mod.getCode().equals(postProcessor.getModuleCode())){
							postProcessModule = mod;
							break;
						}
					}
					
					IntegrationModule module = postProcessModule;
					if(integrationConfiguration != null) {
						postProcessor.prePostProcessShippingQuotes(shippingQuote, packages, orderTotal, delivery, shippingOrigin, store, integrationConfiguration, module, shippingConfiguration, shippingMethods, locale);
					}
				}
			}
			String ipAddress = null;
	    	UserContext context = UserContext.getCurrentInstance();
	    	if(context != null) {
	    		ipAddress = context.getIpAddress();
	    	}
			
			if(shippingQuote!=null && CollectionUtils.isNotEmpty(shippingQuote.getShippingOptions())) {
				//save SHIPPING OPTIONS
				List<ShippingOption> finalShippingOptions = shippingQuote.getShippingOptions();
				for(ShippingOption option : finalShippingOptions) {
					
					//transform to Quote
					Quote q = new Quote();
					q.setCartId(shoppingCartId);
					q.setDelivery(delivery);
					if(!StringUtils.isBlank(ipAddress)) {
						q.setIpAddress(ipAddress);
					}
					if(!StringUtils.isBlank(option.getEstimatedNumberOfDays())) {
						try {
							q.setEstimatedNumberOfDays(Integer.valueOf(option.getEstimatedNumberOfDays()));
						} catch(Exception e) {
							LOGGER.error("Cannot cast to integer " + option.getEstimatedNumberOfDays());
						}
					}
					
					if(freeShipping) {
						q.setFreeShipping(true);
						q.setPrice(new BigDecimal(0));
						q.setModule("FREE");
						q.setOptionCode("FREE");
						q.setOptionName("FREE");
					} else {
						q.setModule(option.getShippingModuleCode());
						q.setOptionCode(option.getOptionCode());
						if(!StringUtils.isBlank(option.getOptionDeliveryDate())) {
							try {
							//q.setOptionDeliveryDate(DateUtil.formatDate(option.getOptionDeliveryDate()));
							} catch(Exception e) {
								LOGGER.error("Cannot transform to date " + option.getOptionDeliveryDate());
							}
						}
						q.setOptionName(option.getOptionName());
						q.setOptionShippingDate(new Date());
						q.setPrice(option.getOptionPrice());
						
					}
					
					if(handlingFees != null) {
						q.setHandling(handlingFees);
					}
					
					q.setQuoteDate(new Date());
					shippingQuoteService.save(q);
					option.setShippingQuoteOptionId(q.getId());
					
				}
			}
			
			
			
		} catch (Exception e) {
			LOGGER.error(e.getMessage(), e);
			throw new ServiceException(e);
		}
		
		return shippingQuote;
		
	}

	@Override
	public List<String> getSupportedCountries(MerchantStore store) throws ServiceException {
		
		List<String> supportedCountries = new ArrayList<String>();
		MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(SUPPORTED_COUNTRIES, store);
		
		if(configuration!=null) {
			
			String countries = configuration.getValue();
			if(!StringUtils.isBlank(countries)) {

				Object objRegions=JSONValue.parse(countries); 
				JSONArray arrayRegions=(JSONArray)objRegions;
				for (Object arrayRegion : arrayRegions) {
					supportedCountries.add((String) arrayRegion);
				}
			}
			
		}
		
		return supportedCountries;
	}
	
	@Override
	public List<Country> getShipToCountryList(MerchantStore store, Language language) throws ServiceException {
		
		
		ShippingConfiguration shippingConfiguration = getShippingConfiguration(store);
		ShippingType shippingType = ShippingType.INTERNATIONAL;
		List<String> supportedCountries = new ArrayList<String>();
		if(shippingConfiguration==null) {
			shippingConfiguration = new ShippingConfiguration();
		}
		
		if(shippingConfiguration.getShippingType()!=null) {
				shippingType = shippingConfiguration.getShippingType();
		}

		
		if(shippingType.name().equals(ShippingType.NATIONAL.name())){
			
			supportedCountries.add(store.getCountry().getIsoCode());
			
		} else {

			MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(SUPPORTED_COUNTRIES, store);
			
			if(configuration!=null) {
				
				String countries = configuration.getValue();
				if(!StringUtils.isBlank(countries)) {

					Object objRegions=JSONValue.parse(countries); 
					JSONArray arrayRegions=(JSONArray)objRegions;
					for (Object arrayRegion : arrayRegions) {
						supportedCountries.add((String) arrayRegion);
					}
				}
				
			}

		}
		
		return countryService.getCountries(supportedCountries, language);

	}
	

	@Override
	public void setSupportedCountries(MerchantStore store, List<String> countryCodes) throws ServiceException {
		
		
		//transform a list of string to json entry
		ObjectMapper mapper = new ObjectMapper();
		
		try {
			String value  = mapper.writeValueAsString(countryCodes);
			
			MerchantConfiguration configuration = merchantConfigurationService.getMerchantConfiguration(SUPPORTED_COUNTRIES, store);
			
			if(configuration==null) {
				configuration = new MerchantConfiguration();
				configuration.
				setKey(SUPPORTED_COUNTRIES);
				configuration.setMerchantStore(store);
			} 
			
			configuration.setValue(value);

			merchantConfigurationService.saveOrUpdate(configuration);
			
		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}
	

	private BigDecimal calculateOrderTotal(List<ShippingProduct> products, MerchantStore store) throws Exception {
		
		BigDecimal total = new BigDecimal(0);
		for(ShippingProduct shippingProduct : products) {
			BigDecimal currentPrice = shippingProduct.getFinalPrice().getFinalPrice();
			currentPrice = currentPrice.multiply(new BigDecimal(shippingProduct.getQuantity()));
			total = total.add(currentPrice);
		}
		
		
		return total;
		
		
	}

	@Override
	public List<PackageDetails> getPackagesDetails(
			List<ShippingProduct> products, MerchantStore store)
			throws ServiceException {
		
		List<PackageDetails> packages = null;
		
		ShippingConfiguration shippingConfiguration = this.getShippingConfiguration(store);
		//determine if the system has to use BOX or ITEM
		ShippingPackageType shippingPackageType = ShippingPackageType.ITEM;
		if(shippingConfiguration!=null) {
			shippingPackageType = shippingConfiguration.getShippingPackageType();
		}
		
		if(shippingPackageType.name().equals(ShippingPackageType.BOX.name())){
			packages = packaging.getBoxPackagesDetails(products, store);
		} else {
			packages = packaging.getItemPackagesDetails(products, store);
		}
		
		return packages;
		
	}

	@Override
	public boolean requiresShipping(List<ShoppingCartItem> items,
			MerchantStore store) throws ServiceException {

		boolean requiresShipping = false;
		for(ShoppingCartItem item : items) {
			Product product = item.getProduct();
			if(!product.isProductVirtual() && product.isProductShipeable()) {
				requiresShipping = true;
			}
		}

		return requiresShipping;		
	}

	@Override
	public ShippingMetaData getShippingMetaData(MerchantStore store)
			throws ServiceException {
		
		
		try {
		
		ShippingMetaData metaData = new ShippingMetaData();

		// configured country
		List<Country> countries = getShipToCountryList(store, store.getDefaultLanguage());
		metaData.setShipToCountry(countries);
		
		// configured modules
		Map<String,IntegrationConfiguration> modules = Optional.ofNullable(getShippingModulesConfigured(store))
				.orElse(Collections.emptyMap());
		metaData.setModules(new ArrayList<>(modules.keySet()));
		
		// pre processors
		List<ShippingQuotePrePostProcessModule> preProcessors = this.shippingModulePreProcessors;
		List<String> preProcessorKeys = new ArrayList<String>();
		if(preProcessors!=null) {
			for(ShippingQuotePrePostProcessModule processor : preProcessors) {
				preProcessorKeys.add(processor.getModuleCode());
				if(SHIPPING_DISTANCE.equals(processor.getModuleCode())) {
					metaData.setUseDistanceModule(true);
				}
			}
		}
		metaData.setPreProcessors(preProcessorKeys);
		
		//post processors
		List<ShippingQuotePrePostProcessModule> postProcessors = this.shippingModulePostProcessors;
		List<String> postProcessorKeys = new ArrayList<String>();
		if(postProcessors!=null) {
			for(ShippingQuotePrePostProcessModule processor : postProcessors) {
				postProcessorKeys.add(processor.getModuleCode());
			}
		}
		metaData.setPostProcessors(postProcessorKeys);
		
		
		return metaData;
		
		} catch(Exception e) {
			throw new ServiceException("Exception while getting shipping metadata ",e);
		}
	}

	@Override
	public boolean hasTaxOnShipping(MerchantStore store) throws ServiceException {
		ShippingConfiguration shippingConfiguration = getShippingConfiguration(store);
		return shippingConfiguration.isTaxOnShipping();
	}
}


```
