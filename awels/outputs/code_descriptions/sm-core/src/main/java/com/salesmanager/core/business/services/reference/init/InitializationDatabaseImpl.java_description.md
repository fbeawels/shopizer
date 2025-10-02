# InitializationDatabaseImpl.java

## Review

## 1. Summary  
The **`InitializationDatabaseImpl`** service is a Spring‑managed component responsible for seeding a freshly installed database with reference data (languages, countries, zones, currencies, security groups, merchant store, modules, etc.).  
Key responsibilities:

| Component | Responsibility |
|-----------|----------------|
| `createSecurityGroups` | Creates a predefined set of `Permission` and `Group` objects, wiring them together via `SecurityGroupsBuilder`. |
| `createLanguages` | Loads ISO language codes from `SchemaConstant.LANGUAGE_ISO_CODE` into the database. |
| `createCountries` | Loads ISO country codes and populates their localized names using the predefined `Locale` map. |
| `createZones` | Reads zone definitions from JSON files (`reference/zoneconfig.json` + per‑language files) and persists them via `ZoneService`. |
| `createCurrencies` | Loads currency definitions from `SchemaConstant.CURRENCY_MAP` (actually only keys are used) and creates a `Currency` entity for each. |
| `createMerchant` | Instantiates the default merchant store, a default tax class, a default manufacturer, and a newsletter opt‑in. |
| `createModules` | Loads integration modules from `reference/integrationmodules.json` and stores them with `ModuleConfigurationService`. |
| `createSubReferences` | Adds a minimal set of catalog reference data (currently only a generic product type). |

The service is annotated with `@Transactional`, meaning all operations within `populate()` run in a single Spring transaction. The code relies heavily on injected services (`ZoneService`, `LanguageService`, `CountryService`, …) and a handful of utility loaders (`ZonesLoader`, `IntegrationModulesLoader`). The design follows a **sequential data‑loading** pattern, executing each step one after another.

---

## 2. Detailed Description  
### Flow of Execution  
1. **Entry Point** – `populate(String contextName)` is called during application startup (typically by an initializer component).  
2. **Initialization** – The `name` field is set for logging context.  
3. **Data Loading** – The method calls, in order:  
   * `createSecurityGroups()`  
   * `createLanguages()`  
   * `createCountries()`  
   * `createZones()`  
   * `createCurrencies()`  
   * `createSubReferences()`  
   * `createModules()`  
   * `createMerchant()`  
4. **Commit** – When the method completes without throwing `ServiceException`, the transaction commits; otherwise it rolls back.

### Assumptions & Constraints  
* The database is **empty** (checked via `isEmpty()`) before calling `populate()`.  
* All `createX()` methods **do not perform existence checks** – they assume no duplicate data will exist.  
* JSON files (`zoneconfig.json`, per‑language zone files, `integrationmodules.json`) are available on the classpath.  
* The `SchemaConstant` map contains valid ISO codes and locales.  
* The service is **single‑threaded** – it is executed once during startup.

### Design Choices  
* **Explicit Service Injection** – Uses JSR‑330 `@Inject` annotations instead of Spring’s `@Autowired`.  
* **Transactional Boundary** – All loading steps share a single transaction, simplifying rollback logic.  
* **Utility Builders** – `SecurityGroupsBuilder` is used to fluently construct groups and permissions.  
* **JSON Loading** – `ZonesLoader` and `IntegrationModulesLoader` encapsulate file parsing, keeping this service focused on persistence.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `public boolean isEmpty()` | Checks if any language exists (`languageService.count() == 0`). | – | `boolean` | – |
| `@Transactional public void populate(String contextName)` | Main entry point; orchestrates all data‑seeding steps. | `contextName` – used only for logging. | `void` | Creates all reference data, throws `ServiceException` on failure. |
| `private void createSecurityGroups()` | Instantiates `Permission` and `Group` objects, wiring them via `SecurityGroupsBuilder`. | – | `void` | Persists permissions and groups. |
| `private void createCurrencies()` | Loops over `SchemaConstant.CURRENCY_MAP` keys, creates a `Currency` entity for each. | – | `void` | Persists currency records. |
| `private void createCountries()` | Loads country ISO codes, persists `Country` and associated `CountryDescription` for each supported language. | – | `void` | Persists countries and descriptions. |
| `private void createZones()` | Reads zone definitions from JSON files, delegating to `addZonesToDb`. | – | `void` | Persists zones and descriptions. |
| `private void addZonesToDb(Map<String, Zone> zonesMap)` | Persists a map of `Zone` objects and their `ZoneDescription`s. | `zonesMap` – key = zone code, value = `Zone`. | `void` | Persists zones; logs but swallows any exception. |
| `private void createLanguages()` | Creates a `Language` entity for each ISO code in `SchemaConstant.LANGUAGE_ISO_CODE`. | – | `void` | Persists languages. |
| `private void createMerchant()` | Instantiates the default `MerchantStore`, `TaxClass`, `Manufacturer`, and a newsletter `Optin`. | – | `void` | Persists merchant and related entities. |
| `private void createModules()` | Loads integration modules from JSON and persists them. | – | `void` | Persists integration modules. |
| `private void createSubReferences()` | Adds catalog reference data (currently only a generic `ProductType`). | – | `void` | Persists product type. |

### Reusable/Utility Methods  
* **`SecurityGroupsBuilder`** – Encapsulates the construction of groups and permissions.  
* **`ZonesLoader` / `IntegrationModulesLoader`** – Abstract JSON deserialization logic, making the service agnostic to file structure.  

---

## 4. Dependencies  

| External Library | Role | Nature |
|------------------|------|--------|
| **Spring Framework** (`@Service`, `@Transactional`, `@Inject`) | DI & transaction management | Third‑party |
| **SLF4J** (`Logger`, `LoggerFactory`) | Logging | Third‑party |
| **Java Standard Library** (`java.util.*`, `java.sql.Date`, `Locale`) | Core utilities | Standard |
| **Salesmanager Core modules** (`com.salesmanager.*`) | Domain models, services, constants | Internal |
| **Jackson / JSON** (implicitly used inside `ZonesLoader`, `IntegrationModulesLoader`) | JSON parsing | Third‑party |

No platform‑specific dependencies; all services are standard Spring beans.  

---

## 5. Additional Notes  

### Strengths  
* **Clear separation of concerns** – each `createX()` method is focused on a single data domain.  
* **Transactional safety** – the whole initialization runs atomically.  
* **Extensibility** – adding new reference data is straightforward: create a new `createX()` method and invoke it in `populate()`.  
* **Use of builders and loaders** – keeps the service lean and testable.

### Weaknesses & Edge Cases  

| Area | Issue | Impact | Recommendation |
|------|-------|--------|----------------|
| **Duplicate data handling** | No checks before creating entities (except via `isEmpty()` for languages). | Subsequent runs on a non‑empty database can cause unique‑constraint violations or duplicate records. | Add existence checks (`service.findByCode(...)`) or catch constraint exceptions and log a warning. |
| **Error handling in `addZonesToDb`** | Swallows all exceptions, only logs them. | A zone parsing error will go unnoticed and may leave the database in an inconsistent state. | Rethrow a checked exception (e.g., `ServiceException`) or at least propagate the cause to trigger transaction rollback. |
| **Hard‑coded strings** | Many literals (e.g., `"Shopizer"`, `"localhost:8080"`, `"DECEMBER"`, etc.) embedded in code. | Difficult to change configuration without recompilation. | Externalize to a properties file or `SchemaConstant`. |
| **Redundant permission creation** | Repeated `permissionService.create(...)` calls without deduplication. | Unnecessary DB writes if permissions already exist. | Centralize permission creation in a helper that checks for existence first. |
| **Large JSON handling** | Reading zone files into memory as whole maps. | Possible memory pressure if zone data grows large. | Stream parsing or load only necessary parts. |
| **Use of `java.sql.Date`** | Mixing `java.sql.Date` with `java.util.Date` in code. | Potential timezone or precision issues. | Prefer `java.time.LocalDate` or `Instant`. |
| **`createMerchant` hardcodes merchant details** | The default merchant is always the same. | Not suitable for multi‑tenant or environment‑specific setups. | Accept a `MerchantStore` DTO or read from config. |
| **No logging of success** | Only logs at the start of each step. | Hard to audit which step succeeded or failed. | Log at the end of each method (e.g., `LOGGER.info("Created X languages")`). |

### Suggested Enhancements  

1. **Idempotency** – Wrap each `createX()` in a check to avoid duplicate inserts.  
2. **Refactor permission/group creation** – Use arrays/lists and a loop instead of repetitive code.  
3. **Central configuration** – Move all literal values into `SchemaConstant` or Spring `@ConfigurationProperties`.  
4. **Better exception propagation** – Replace the silent catch in `addZonesToDb` with a rethrow of `ServiceException`.  
5. **Unit testing** – Write tests that load a mock database, invoke `populate()`, and verify that all expected records exist.  
6. **Asynchronous loading** – For large datasets, consider loading zones and modules asynchronously after the main transaction.  
7. **Logging improvements** – Add granular logging, e.g., number of records created.  

### Future Extensions  

* **Multi‑language zone descriptions** – Load descriptions from a separate resource bundle instead of JSON.  
* **Dynamic merchant creation** – Expose a REST endpoint to seed a merchant for a new store.  
* **Plugin system** – Allow custom loaders to add new reference data types without modifying this class.  

---

### Bottom Line  
`InitializationDatabaseImpl` is a well‑structured, domain‑centric data seeding service. It leverages Spring transaction management, dependency injection, and utility loaders to keep the logic concise. However, the current implementation is **not idempotent** and contains several hard‑coded values and silent error paths that could lead to silent failures or duplicate data on subsequent runs. Addressing these concerns with existence checks, better exception propagation, and externalized configuration would make the initializer robust and production‑ready.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.reference.init;

import java.sql.Date;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import javax.inject.Inject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.product.manufacturer.ManufacturerService;
import com.salesmanager.core.business.services.catalog.product.type.ProductTypeService;
import com.salesmanager.core.business.services.merchant.MerchantStoreService;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.currency.CurrencyService;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.business.services.reference.loader.IntegrationModulesLoader;
import com.salesmanager.core.business.services.reference.loader.ZonesLoader;
import com.salesmanager.core.business.services.reference.zone.ZoneService;
import com.salesmanager.core.business.services.system.ModuleConfigurationService;
import com.salesmanager.core.business.services.system.optin.OptinService;
import com.salesmanager.core.business.services.tax.TaxClassService;
import com.salesmanager.core.business.services.user.GroupService;
import com.salesmanager.core.business.services.user.PermissionService;
import com.salesmanager.core.business.utils.SecurityGroupsBuilder;
import com.salesmanager.core.constants.SchemaConstant;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.manufacturer.ManufacturerDescription;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.country.CountryDescription;
import com.salesmanager.core.model.reference.currency.Currency;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.core.model.reference.zone.ZoneDescription;
import com.salesmanager.core.model.system.IntegrationModule;
import com.salesmanager.core.model.system.optin.Optin;
import com.salesmanager.core.model.system.optin.OptinType;
import com.salesmanager.core.model.tax.taxclass.TaxClass;
import com.salesmanager.core.model.user.Group;
import com.salesmanager.core.model.user.GroupType;
import com.salesmanager.core.model.user.Permission;

@Service("initializationDatabase")
public class InitializationDatabaseImpl implements InitializationDatabase {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(InitializationDatabaseImpl.class);
	

	@Inject
	private ZoneService zoneService;
	
	@Inject
	private LanguageService languageService;
	
	@Inject
	private CountryService countryService;
	
	@Inject
	private CurrencyService currencyService;
	
	@Inject
	protected MerchantStoreService merchantService;
		
	@Inject
	protected ProductTypeService productTypeService;
	
	@Inject
	private TaxClassService taxClassService;
	
	@Inject
	private ZonesLoader zonesLoader;
	
	@Inject
	private IntegrationModulesLoader modulesLoader;
	
	@Inject
	private ManufacturerService manufacturerService;
	
	@Inject
	private ModuleConfigurationService moduleConfigurationService;
	
	@Inject
	private OptinService optinService;
	
	@Inject
	protected GroupService   groupService;
	
	@Inject
	protected PermissionService   permissionService;

	private String name;
	
	public boolean isEmpty() {
		return languageService.count() == 0;
	}
	
	@Transactional
	public void populate(String contextName) throws ServiceException {
		this.name =  contextName;
		
		createSecurityGroups();
		createLanguages();
		createCountries();
		createZones();
		createCurrencies();
		createSubReferences();
		createModules();
		createMerchant();


	}
	
	private void createSecurityGroups() throws ServiceException {
		
		  //create permissions
		  //Map name object
		  Map<String, Permission> permissionKeys = new HashMap<String, Permission>();
		  Permission AUTH = new Permission("AUTH");
		  permissionService.create(AUTH);
		  permissionKeys.put(AUTH.getPermissionName(), AUTH);
		  
		  Permission SUPERADMIN = new Permission("SUPERADMIN");
		  permissionService.create(SUPERADMIN);
		  permissionKeys.put(SUPERADMIN.getPermissionName(), SUPERADMIN);
		  
		  Permission ADMIN = new Permission("ADMIN");
		  permissionService.create(ADMIN);
		  permissionKeys.put(ADMIN.getPermissionName(), ADMIN);
		  
		  Permission PRODUCTS = new Permission("PRODUCTS");
		  permissionService.create(PRODUCTS);
		  permissionKeys.put(PRODUCTS.getPermissionName(), PRODUCTS);
		  
		  Permission ORDER = new Permission("ORDER");
		  permissionService.create(ORDER);
		  permissionKeys.put(ORDER.getPermissionName(), ORDER);
		  
		  Permission CONTENT = new Permission("CONTENT");
		  permissionService.create(CONTENT);
		  permissionKeys.put(CONTENT.getPermissionName(), CONTENT);
		  
		  Permission STORE = new Permission("STORE");
		  permissionService.create(STORE);
		  permissionKeys.put(STORE.getPermissionName(), STORE);
		  
		  Permission TAX = new Permission("TAX");
		  permissionService.create(TAX);
		  permissionKeys.put(TAX.getPermissionName(), TAX);
		  
		  Permission PAYMENT = new Permission("PAYMENT");
		  permissionService.create(PAYMENT);
		  permissionKeys.put(PAYMENT.getPermissionName(), PAYMENT);
		  
		  Permission CUSTOMER = new Permission("CUSTOMER");
		  permissionService.create(CUSTOMER);
		  permissionKeys.put(CUSTOMER.getPermissionName(), CUSTOMER);
		  
		  Permission SHIPPING = new Permission("SHIPPING");
		  permissionService.create(SHIPPING);
		  permissionKeys.put(SHIPPING.getPermissionName(), SHIPPING);
		  
		  Permission AUTH_CUSTOMER = new Permission("AUTH_CUSTOMER");
		  permissionService.create(AUTH_CUSTOMER);
		  permissionKeys.put(AUTH_CUSTOMER.getPermissionName(), AUTH_CUSTOMER);
		
		  SecurityGroupsBuilder groupBuilder = new SecurityGroupsBuilder();
		  groupBuilder
		  .addGroup("SUPERADMIN", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("SUPERADMIN"))
		  .addPermission(permissionKeys.get("ADMIN"))
		  .addPermission(permissionKeys.get("PRODUCTS"))
		  .addPermission(permissionKeys.get("ORDER"))
		  .addPermission(permissionKeys.get("CONTENT"))
		  .addPermission(permissionKeys.get("STORE"))
		  .addPermission(permissionKeys.get("TAX"))
		  .addPermission(permissionKeys.get("PAYMENT"))
		  .addPermission(permissionKeys.get("CUSTOMER"))
		  .addPermission(permissionKeys.get("SHIPPING"))
		  
		  .addGroup("ADMIN", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("ADMIN"))
		  .addPermission(permissionKeys.get("PRODUCTS"))
		  .addPermission(permissionKeys.get("ORDER"))
		  .addPermission(permissionKeys.get("CONTENT"))
		  .addPermission(permissionKeys.get("STORE"))
		  .addPermission(permissionKeys.get("TAX"))
		  .addPermission(permissionKeys.get("PAYMENT"))
		  .addPermission(permissionKeys.get("CUSTOMER"))
		  .addPermission(permissionKeys.get("SHIPPING"))
		  
		  .addGroup("ADMIN_RETAILER", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("ADMIN"))
		  .addPermission(permissionKeys.get("PRODUCTS"))
		  .addPermission(permissionKeys.get("ORDER"))
		  .addPermission(permissionKeys.get("CONTENT"))
		  .addPermission(permissionKeys.get("STORE"))
		  .addPermission(permissionKeys.get("TAX"))
		  .addPermission(permissionKeys.get("PAYMENT"))
		  .addPermission(permissionKeys.get("CUSTOMER"))
		  .addPermission(permissionKeys.get("SHIPPING"))
		  
		  .addGroup("ADMIN_STORE", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("CONTENT"))
		  .addPermission(permissionKeys.get("STORE"))
		  .addPermission(permissionKeys.get("TAX"))
		  .addPermission(permissionKeys.get("PAYMENT"))
		  .addPermission(permissionKeys.get("CUSTOMER"))
		  .addPermission(permissionKeys.get("SHIPPING"))
		  
		  .addGroup("ADMIN_CATALOGUE", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("PRODUCTS"))
		  
		  .addGroup("ADMIN_ORDER", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("ORDER"))
		  
		  .addGroup("ADMIN_CONTENT", GroupType.ADMIN)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("CONTENT"))
		  
		  .addGroup("CUSTOMER", GroupType.CUSTOMER)
		  .addPermission(permissionKeys.get("AUTH"))
		  .addPermission(permissionKeys.get("AUTH_CUSTOMER"));
		  
		  for(Group g : groupBuilder.build()) {
			  groupService.create(g);
		  }

		
	}
	


	private void createCurrencies() throws ServiceException {
		LOGGER.info(String.format("%s : Populating Currencies ", name));

		for (String code : SchemaConstant.CURRENCY_MAP.keySet()) {
  
            try {
            	java.util.Currency c = java.util.Currency.getInstance(code);
            	
            	if(c==null) {
            		LOGGER.info(String.format("%s : Populating Currencies : no currency for code : %s", name, code));
            	}
            	
            		//check if it exist
            		
	            	Currency currency = new Currency();
	            	currency.setName(c.getCurrencyCode());
	            	currency.setCurrency(c);
	            	currencyService.create(currency);

            //System.out.println(l.getCountry() + "   " + c.getSymbol() + "  " + c.getSymbol(l));
            } catch (IllegalArgumentException e) {
            	LOGGER.info(String.format("%s : Populating Currencies : no currency for code : %s", name, code));
            }
        }  
	}

	private void createCountries() throws ServiceException {
		LOGGER.info(String.format("%s : Populating Countries ", name));
		List<Language> languages = languageService.list();
		for(String code : SchemaConstant.COUNTRY_ISO_CODE) {
			Locale locale = SchemaConstant.LOCALES.get(code);
			if (locale != null) {
				Country country = new Country(code);
				countryService.create(country);
				
				for (Language language : languages) {
					String name = locale.getDisplayCountry(new Locale(language.getCode()));
					//byte[] ptext = value.getBytes(Constants.ISO_8859_1); 
					//String name = new String(ptext, Constants.UTF_8); 
					CountryDescription description = new CountryDescription(language, name);
					countryService.addCountryDescription(country, description);
				}
			}
		}
	}
	
	private void createZones() throws ServiceException {
		LOGGER.info(String.format("%s : Populating Zones ", name));
        try {

    		  Map<String,Zone> zonesMap = new HashMap<String,Zone>();
    		  zonesMap = zonesLoader.loadZones("reference/zoneconfig.json");
    		  
    		  this.addZonesToDb(zonesMap);
/*              
              for (Map.Entry<String, Zone> entry : zonesMap.entrySet()) {
            	    String key = entry.getKey();
            	    Zone value = entry.getValue();
            	    if(value.getDescriptions()==null) {
            	    	LOGGER.warn("This zone " + key + " has no descriptions");
            	    	continue;
            	    }
            	    
            	    List<ZoneDescription> zoneDescriptions = value.getDescriptions();
            	    value.setDescriptons(null);

            	    zoneService.create(value);
            	    
            	    for(ZoneDescription description : zoneDescriptions) {
            	    	description.setZone(value);
            	    	zoneService.addDescription(value, description);
            	    }
              }*/
              
              //lookup additional zones
              //iterate configured languages
      		  LOGGER.info("Populating additional zones");

              //load reference/zones/* (zone config for additional country)
              //example in.json and in-fr.son
              //will load es zones and use a specific file for french es zones
      		  List<Map<String, Zone>> loadIndividualZones = zonesLoader.loadIndividualZones();
      		  
      		loadIndividualZones.forEach(this::addZonesToDb);

  		} catch (Exception e) {
  		    
  			throw new ServiceException(e);
  		}

	}

	
	private void addZonesToDb(Map<String,Zone> zonesMap) throws RuntimeException {
		
		try {
		
	        for (Map.Entry<String, Zone> entry : zonesMap.entrySet()) {
	    	    String key = entry.getKey();
	    	    Zone value = entry.getValue();

	    	    if(value.getDescriptions()==null) {
	    	    	LOGGER.warn("This zone " + key + " has no descriptions");
	    	    	continue;
	    	    }
	    	    
	    	    List<ZoneDescription> zoneDescriptions = value.getDescriptions();
	    	    value.setDescriptons(null);
	
	    	    zoneService.create(value);
	    	    
	    	    for(ZoneDescription description : zoneDescriptions) {
	    	    	description.setZone(value);
	    	    	zoneService.addDescription(value, description);
	    	    }
	        }
        
		}catch(Exception e) {
			LOGGER.error("An error occured while loading zones",e);
			
		}
		
	}
	
	private void createLanguages() throws ServiceException {
		LOGGER.info(String.format("%s : Populating Languages ", name));
		for(String code : SchemaConstant.LANGUAGE_ISO_CODE) {
			Language language = new Language(code);
			languageService.create(language);
		}
	}
	
	private void createMerchant() throws ServiceException {
		LOGGER.info(String.format("%s : Creating merchant ", name));
		
		Date date = new Date(System.currentTimeMillis());
		
		Language en = languageService.getByCode("en");
		Country ca = countryService.getByCode("CA");
		Currency currency = currencyService.getByCode("CAD");
		Zone qc = zoneService.getByCode("QC");
		
		List<Language> supportedLanguages = new ArrayList<Language>();
		supportedLanguages.add(en);
		
		//create a merchant
		MerchantStore store = new MerchantStore();
		store.setCountry(ca);
		store.setCurrency(currency);
		store.setDefaultLanguage(en);
		store.setInBusinessSince(date);
		store.setZone(qc);
		store.setStorename("Shopizer");
		store.setStorephone("888-888-8888");
		store.setCode(MerchantStore.DEFAULT_STORE);
		store.setStorecity("My city");
		store.setStoreaddress("1234 Street address");
		store.setStorepostalcode("H2H-2H2");
		store.setStoreEmailAddress("contact@shopizer.com");
		store.setDomainName("localhost:8080");
		store.setStoreTemplate("december");
		store.setRetailer(true);
		store.setLanguages(supportedLanguages);
		
		merchantService.create(store);
		
		
		TaxClass taxclass = new TaxClass(TaxClass.DEFAULT_TAX_CLASS);
		taxclass.setMerchantStore(store);
		
		taxClassService.create(taxclass);
		
		//create default manufacturer
		Manufacturer defaultManufacturer = new Manufacturer();
		defaultManufacturer.setCode("DEFAULT");
		defaultManufacturer.setMerchantStore(store);
		
		ManufacturerDescription manufacturerDescription = new ManufacturerDescription();
		manufacturerDescription.setLanguage(en);
		manufacturerDescription.setName("DEFAULT");
		manufacturerDescription.setManufacturer(defaultManufacturer);
		manufacturerDescription.setDescription("DEFAULT");
		defaultManufacturer.getDescriptions().add(manufacturerDescription);
		
		manufacturerService.create(defaultManufacturer);
		
	   Optin newsletter = new Optin();
	   newsletter.setCode(OptinType.NEWSLETTER.name());
	   newsletter.setMerchant(store);
	   newsletter.setOptinType(OptinType.NEWSLETTER);
	   optinService.create(newsletter);
		
		
	}

	private void createModules() throws ServiceException {
		
		try {
			
			List<IntegrationModule> modules = modulesLoader.loadIntegrationModules("reference/integrationmodules.json");
            for (IntegrationModule entry : modules) {
        	    moduleConfigurationService.create(entry);
          }
			
			
		} catch (Exception e) {
			throw new ServiceException(e);
		}
		
		
	}
	
	private void createSubReferences() throws ServiceException {
		
		LOGGER.info(String.format("%s : Loading catalog sub references ", name));
		
		
		ProductType productType = new ProductType();
		productType.setCode(ProductType.GENERAL_TYPE);
		productTypeService.create(productType);


		
		
	}
	

	



}



```
