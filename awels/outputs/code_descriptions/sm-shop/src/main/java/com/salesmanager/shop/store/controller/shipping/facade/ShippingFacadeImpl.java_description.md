# ShippingFacadeImpl.java

## Review

## 1. Summary  

**Purpose**  
`ShippingFacadeImpl` is a Spring service that exposes shipping‑related operations for the storefront layer of the application. It acts as a façade over several domain services (shipping, country, zone, origin) and translates between API DTOs (`ExpeditionConfiguration`, `PackageDetails`, `ReadableAddress`, etc.) and the core domain model (`ShippingConfiguration`, `Package`, `ShippingOrigin`, etc.).  

**Key Components**  

| Component | Role |
|-----------|------|
| `ShippingService` | Persists and retrieves global shipping configuration, manages supported countries. |
| `ShippingOriginService` | Handles CRUD for the store’s shipping origin address. |
| `CountryService`, `ZoneService` | Resolve reference data by code. |
| DTOs (`ExpeditionConfiguration`, `PackageDetails`, `ReadableAddress`, `ReadableCountry`) | Represent data structures exposed to the API. |
| `ReadableCountryPopulator` | Converts core `Country` objects into `ReadableCountry`. |

**Design Patterns / Frameworks**  

* **Facade Pattern** – Exposes a simplified API to the storefront layer.  
* **Adapter/Populator Pattern** – The `ReadableCountryPopulator` converts between model and DTO.  
* **Spring DI** – Dependencies are injected via `@Autowired`.  
* **Java Streams** – Used for filtering, mapping, sorting, and collecting.  

---

## 2. Detailed Description  

### Initialization  

* Spring creates a singleton `ShippingFacadeImpl` bean and injects the four service dependencies.  
* No explicit constructor logic is required; all business logic resides in public methods.

### Runtime Flow  

1. **Expedition Configuration**  
   * `getExpeditionConfiguration()` reads the current configuration from the DB, determines if international shipping is enabled, flags tax usage, and builds a sorted list of supported country codes.  
   * `saveExpeditionConfiguration()` updates the configuration and pushes the list of supported country codes to `ShippingService`.  

2. **Shipping Origin**  
   * `getShippingOrigin()` retrieves the current origin for the store and maps it to a `ReadableAddress`.  
   * `saveShippingOrigin()` persists a new/updated origin, resolving the country and zone references.  

3. **Package Management**  
   * `createPackage()`, `getPackage()`, `listPackages()`, `updatePackage()`, `deletePackage()` all operate on the `packages` collection inside `ShippingConfiguration`.  
   * The helper methods `toPackage()` / `toPackageDetails()` perform conversions between DTO and domain objects.  

4. **Ship‑To Countries**  
   * `shipToCountry()` asks `ShippingService` for a list of countries that the store can ship to, then converts each to `ReadableCountry` using `ReadableCountryPopulator`.  

### Cleanup  

No explicit resource cleanup is performed; the service relies on Spring to manage bean lifecycle.

### Assumptions & Constraints  

* A `ShippingConfiguration` exists per store; if not found, a default with `INTERNATIONAL` type is created.  
* Country codes and zone codes are assumed to be valid and exist in reference tables.  
* No concurrency control is implemented for package manipulation – potential race conditions if multiple threads update the same configuration simultaneously.  

### Architecture  

The façade sits above the domain services and keeps the API layer decoupled from the persistence layer. Business logic is largely delegated to the underlying services; the façade mainly handles DTO conversion, validation, and error handling.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getExpeditionConfiguration(MerchantStore, Language)` | Builds an `ExpeditionConfiguration` DTO from the DB config and supported countries. | `store`, `language` | `ExpeditionConfiguration` | None |
| `saveExpeditionConfiguration(ExpeditionConfiguration, MerchantStore)` | Persists a new configuration and updates supported countries. | `expedition`, `store` | `void` | DB update |
| `saveShippingConfiguration(ShippingConfiguration, MerchantStore)` | Wrapper for `ShippingService.saveShippingConfiguration`. | `config`, `store` | `void` | DB update |
| `getShippingOrigin(MerchantStore)` | Retrieves the store’s shipping origin as a `ReadableAddress`. | `store` | `ReadableAddress` | None |
| `saveShippingOrigin(PersistableAddress, MerchantStore)` | Persists the shipping origin, resolving country/zone. | `address`, `store` | `void` | DB update |
| `getDbConfig(MerchantStore)` | Loads or creates a default `ShippingConfiguration`. | `store` | `ShippingConfiguration` | None |
| `createPackage(PackageDetails, MerchantStore)` | Adds a new package to the config after uniqueness check. | `packaging`, `store` | `void` | DB update |
| `packageExists(ShippingConfiguration, PackageDetails)` | Checks if a package code already exists. | `config`, `packageDetails` | `boolean` | None |
| `packageDetails(ShippingConfiguration, String)` | Retrieves a package by code. | `config`, `code` | `Package` or `null` | None |
| `getPackage(String, MerchantStore)` | Gets a package DTO by code. | `code`, `store` | `PackageDetails` | None |
| `listPackages(MerchantStore)` | Lists all package DTOs for a store. | `store` | `List<PackageDetails>` | None |
| `updatePackage(String, PackageDetails, MerchantStore)` | Replaces an existing package with new data. | `code`, `packaging`, `store` | `void` | DB update |
| `deletePackage(String, MerchantStore)` | Removes a package from the config. | `code`, `store` | `void` | DB update |
| `toPackageDetails(com.salesmanager.core.model.shipping.Package)` | Converts domain package to DTO. | `pack` | `PackageDetails` | None |
| `toPackage(PackageDetails)` | Converts DTO to domain package. | `pack` | `Package` | None |
| `shipToCountry(MerchantStore, Language)` | Retrieves a sorted list of countries the store can ship to. | `store`, `language` | `List<ReadableCountry>` | None |
| `convert(Country, MerchantStore, Language)` | Helper that uses `ReadableCountryPopulator`. | `country`, `store`, `lang` | `ReadableCountry` | None |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.stereotype.Service` | Spring | Marks class as a service bean. |
| `org.springframework.beans.factory.annotation.Autowired` | Spring | Injects dependent services. |
| `org.slf4j.Logger` | SLF4J | Logging. |
| `org.apache.commons.collections4.CollectionUtils` | Commons Collections | Utility for empty checks. |
| `org.jsoup.helper.Validate` | Jsoup | Parameter validation. |
| `com.salesmanager.core.business.services.*` | Internal | Core domain services. |
| `com.salesmanager.core.model.*` | Internal | Domain model entities. |
| `com.salesmanager.shop.model.*` | Internal | DTOs exposed to the storefront. |
| `com.salesmanager.shop.populator.references.ReadableCountryPopulator` | Internal | Converter. |
| `com.salesmanager.shop.store.api.exception.*` | Internal | Runtime exceptions specific to the API layer. |

All dependencies are either Spring framework components or internal project modules; no external third‑party libraries beyond Apache Commons and Jsoup are used.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation of concerns** – DTO conversion is isolated from business logic.  
* **Consistent error handling** – Wraps checked `ServiceException` into unchecked runtime exceptions with descriptive messages.  
* **Use of Java Streams** – Concise and readable collection operations.  

### Potential Issues & Edge Cases  

1. **Thread Safety** – The service manipulates in‑memory collections (`config.getPackages()`) and then persists the entire config. If two threads concurrently create or update packages for the same store, one update may overwrite the other. Consider synchronizing on the store or using optimistic locking.  

2. **Null Country/Zone Handling** – In `saveShippingOrigin()`, `zoneService.getByCode()` may return `null`, leading to the fallback to `address.getStateProvince()`. If the state code is also invalid, the origin may be persisted with an incomplete address. Adding validation or fallback to a default zone could improve robustness.  

3. **Validation of Numeric Fields** – `PackageDetails` fields like height, width, weight are set directly from DTO without range checks. Malformed data could lead to invalid shipping packages.  

4. **Exception Messaging** – Some messages duplicate the same text (“Error while getting expedition configuration”). Unique identifiers (store code, package code) help with debugging.  

5. **Internationalization** – The service accepts a `Language` parameter but does not use it for configuration or country lists beyond `ReadableCountryPopulator`. Ensure that `ReadableCountryPopulator` correctly handles the language.  

6. **Logging** – Sensitive data (e.g., addresses) should not be logged. Current logs only record error traces.  

### Future Enhancements  

* **Optimistic Locking** – Add a version field to `ShippingConfiguration` to prevent lost updates.  
* **Validation Layer** – Use a dedicated validator for DTOs (e.g., Hibernate Validator) instead of manual `Validate` checks.  
* **Unit Tests** – Add comprehensive tests covering package CRUD, origin handling, and edge cases.  
* **Cache** – Cache frequently accessed configurations or country lists to reduce DB load.  
* **API Exposure** – Expose the façade via REST controllers or GraphQL resolvers.  

Overall, `ShippingFacadeImpl` is well‑structured and follows standard Spring service conventions. Addressing concurrency and validation concerns will improve reliability in a production environment.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.shipping.facade;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.stream.Collectors;

import org.apache.commons.collections4.CollectionUtils;
import org.jsoup.helper.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ConversionException;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.zone.ZoneService;
import com.salesmanager.core.business.services.shipping.ShippingOriginService;
import com.salesmanager.core.business.services.shipping.ShippingService;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingOrigin;
import com.salesmanager.core.model.shipping.ShippingPackageType;
import com.salesmanager.core.model.shipping.ShippingType;
import com.salesmanager.shop.model.references.PersistableAddress;
import com.salesmanager.shop.model.references.ReadableAddress;
import com.salesmanager.shop.model.references.ReadableCountry;
import com.salesmanager.shop.model.shipping.ExpeditionConfiguration;
import com.salesmanager.shop.populator.references.ReadableCountryPopulator;
import com.salesmanager.shop.store.api.exception.ConversionRuntimeException;
import com.salesmanager.shop.store.api.exception.OperationNotAllowedException;
import com.salesmanager.shop.store.api.exception.ResourceNotFoundException;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;

@Service("shippingFacade")
public class ShippingFacadeImpl implements ShippingFacade {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ShippingFacadeImpl.class);

	@Autowired
	ShippingOriginService shippingOriginService;
	
	@Autowired
	ShippingService shippingService;
	
	@Autowired
	CountryService countryService;
	
	@Autowired
	ZoneService zoneService;
	
	


	@Override
	public ExpeditionConfiguration getExpeditionConfiguration(MerchantStore store, Language language) {
		ExpeditionConfiguration expeditionConfiguration = new ExpeditionConfiguration();
		try {
			
			ShippingConfiguration config = getDbConfig(store);
			if(config!=null) {
				expeditionConfiguration.setIternationalShipping(config.getShipType()!=null && config.getShipType().equals(ShippingType.INTERNATIONAL.name())?true:false);
				expeditionConfiguration.setTaxOnShipping(config.isTaxOnShipping());
			}
			
			List<String> countries = shippingService.getSupportedCountries(store);

			if(!CollectionUtils.isEmpty(countries)) {
				
				List<String> countryCode = countries.stream()
						.sorted(Comparator.comparing(n->n.toString()))
						.collect(Collectors.toList());
				
				expeditionConfiguration.setShipToCountry(countryCode);
			}

		} catch (ServiceException e) {
			LOGGER.error("Error while getting expedition configuration", e);
			throw new ServiceRuntimeException("Error while getting Expedition configuration for store[" + store.getCode() + "]", e);
		}
		return expeditionConfiguration;
	}

	@Override
	public void saveExpeditionConfiguration(ExpeditionConfiguration expedition, MerchantStore store) {
		Validate.notNull(expedition, "ExpeditionConfiguration cannot be null");
		try {
			
			//get original configuration
			ShippingConfiguration config = getDbConfig(store);
			config.setTaxOnShipping(expedition.isTaxOnShipping());
			config.setShippingType(expedition.isIternationalShipping()?ShippingType.INTERNATIONAL:ShippingType.NATIONAL);
			this.saveShippingConfiguration(config, store);
			
			shippingService.setSupportedCountries(store, expedition.getShipToCountry());


		} catch (ServiceException e) {
			LOGGER.error("Error while getting expedition configuration", e);
			throw new ServiceRuntimeException("Error while getting Expedition configuration for store[" + store.getCode() + "]", e);
		}

	}
	
	private void saveShippingConfiguration(ShippingConfiguration config, MerchantStore store) throws ServiceRuntimeException {
		try {
			shippingService.saveShippingConfiguration(config, store);
		} catch (ServiceException e) {
			LOGGER.error("Error while saving shipping configuration", e);
			throw new ServiceRuntimeException("Error while saving shipping configuration for store [" + store.getCode() + "]", e);
		}
	}

	@Override
	public ReadableAddress getShippingOrigin(MerchantStore store) {
		
		ShippingOrigin o = shippingOriginService.getByStore(store);
		
		if(o == null) {
			throw new ResourceNotFoundException("Shipping origin does not exists for store [" + store.getCode() + "]");
		}
		
		ReadableAddress address = new ReadableAddress();
		address.setAddress(o.getAddress());
		address.setActive(o.isActive());
		address.setCity(o.getCity());
		address.setPostalCode(o.getPostalCode());
		if(o.getCountry()!=null) {
			address.setCountry(o.getCountry().getIsoCode());
		}
		Zone z = o.getZone();
		if(z != null) {
			address.setStateProvince(z.getCode());
		} else {
			address.setStateProvince(o.getState());
		}

		return address;
	}

	@Override
	public void saveShippingOrigin(PersistableAddress address, MerchantStore store) {
		Validate.notNull(address, "PersistableAddress cannot be null");
		try {
			ShippingOrigin o = shippingOriginService.getByStore(store);
			if(o == null) {
				o = new ShippingOrigin();
			}
			
			o.setAddress(address.getAddress());
			o.setCity(address.getCity());
			o.setCountry(countryService.getByCode(address.getCountry()));
			o.setMerchantStore(store);
			o.setActive(address.isActive());
			o.setPostalCode(address.getPostalCode());
			
			Zone zone = zoneService.getByCode(address.getStateProvince());
			if(zone == null) {
				o.setState(address.getStateProvince());
			} else {
				o.setZone(zone);
			}
			
			shippingOriginService.save(o);
			
		} catch (ServiceException e) {
			LOGGER.error("Error while getting shipping origin for country [" + address.getCountry() + "]",e);
			throw new ServiceRuntimeException("Error while getting shipping origin for country [" + address.getCountry() + "]",e);
		}


	}

	private ShippingConfiguration getDbConfig(MerchantStore store) {

		try {
			//get original configuration
			ShippingConfiguration config = shippingService.getShippingConfiguration(store);
			if(config==null) {
				config = new ShippingConfiguration();
				config.setShippingType(ShippingType.INTERNATIONAL);
			}

			return config;
		} catch (ServiceException e) {
			LOGGER.error("Error while getting expedition configuration", e);
			throw new ServiceRuntimeException("Error while getting Expedition configuration for store[" + store.getCode() + "]", e);
		}
		
	}

	@Override
	public void createPackage(PackageDetails packaging, MerchantStore store) {
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notNull(packaging, "PackageDetails cannot be null");
		ShippingConfiguration config = getDbConfig(store);
		
		if(this.packageExists(config, packaging)) {
			throw new OperationNotAllowedException("Package with unique code [" + packaging.getCode() + "] already exist");
		}
		
		com.salesmanager.core.model.shipping.Package pack = toPackage(packaging);
		
		
		//need to check if code exists
		config.getPackages().add(pack);
		this.saveShippingConfiguration(config, store);
	
	}
	
	private boolean packageExists(ShippingConfiguration configuration, PackageDetails packageDetails) {
		
		Validate.notNull(configuration,"ShippingConfiguration cannot be null");
		Validate.notNull(packageDetails, "PackageDetails cannot be null");
		Validate.notEmpty(packageDetails.getCode(), "PackageDetails code cannot be empty");
		
		List<com.salesmanager.core.model.shipping.Package> packages = configuration.getPackages().stream().filter(p -> p.getCode().equalsIgnoreCase(packageDetails.getCode())).collect(Collectors.toList());
		
		if(packages.isEmpty()) {
			return false;
		} else {
			return true;
		}
		
		
	}
	
	private com.salesmanager.core.model.shipping.Package packageDetails(ShippingConfiguration configuration, String code) {
		
		Validate.notNull(configuration,"ShippingConfiguration cannot be null");
		Validate.notNull(code, "PackageDetails code cannot be null");

		List<com.salesmanager.core.model.shipping.Package> packages = configuration.getPackages().stream().filter(p -> p.getCode().equalsIgnoreCase(code)).collect(Collectors.toList());
		
		if(!packages.isEmpty()) {
			return packages.get(0);
		} else {
			return null;
		}

		
	}

	@Override
	public PackageDetails getPackage(String code, MerchantStore store) {
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notEmpty(code,"Packaging unique code cannot be empty");
		
		ShippingConfiguration config = getDbConfig(store);
		
		com.salesmanager.core.model.shipping.Package p = this.packageDetails(config, code);
		
		if(p == null) {
			throw new ResourceNotFoundException("Package with unique code [" + code + "] not found");
		}
		
		return toPackageDetails(p);
	}

	@Override
	public List<PackageDetails> listPackages(MerchantStore store) {
		Validate.notNull(store, "MerchantStore cannot be null");
		ShippingConfiguration config = getDbConfig(store);
		
		return config.getPackages().stream().map(p -> this.toPackageDetails(p)).collect(Collectors.toList());

	}

	@Override
	public void updatePackage(String code, PackageDetails packaging, MerchantStore store) {
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notNull(packaging, "PackageDetails cannot be null");
		Validate.notEmpty(code,"Packaging unique code cannot be empty");
		
		ShippingConfiguration config = getDbConfig(store);
		
		com.salesmanager.core.model.shipping.Package p = this.packageDetails(config, code);
		
		if(p == null) {
			throw new ResourceNotFoundException("Package with unique code [" + packaging.getCode() + "] not found");
		}
		
		com.salesmanager.core.model.shipping.Package pack = toPackage(packaging);
		pack.setCode(code);
		
		//need to check if code exists
		List<com.salesmanager.core.model.shipping.Package> packs = config.getPackages().stream().filter(pa -> !pa.getCode().equals(code)).collect(Collectors.toList());
		packs.add(pack);
		
		config.setPackages(packs);
		this.saveShippingConfiguration(config, store);
		
	}

	@Override
	public void deletePackage(String code, MerchantStore store) {
		
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notEmpty(code,"Packaging unique code cannot be empty");
		
		ShippingConfiguration config = getDbConfig(store);
		
		List<com.salesmanager.core.model.shipping.Package> packages = config.getPackages();
		
		List<com.salesmanager.core.model.shipping.Package> packList = config.getPackages().stream().filter(p -> p.getCode().equalsIgnoreCase(code)).collect(Collectors.toList());
		
		if(!packList.isEmpty()) {
			packages.removeAll(packList);
			config.setPackages(packages);
			this.saveShippingConfiguration(config, store);
		} 
		
	}
	
	private PackageDetails toPackageDetails(com.salesmanager.core.model.shipping.Package pack) {
		PackageDetails details = new PackageDetails();
		details.setCode(pack.getCode());
		details.setShippingHeight(pack.getBoxHeight());
		details.setShippingLength(pack.getBoxLength());
		details.setShippingMaxWeight(pack.getMaxWeight());
		//details.setShippingQuantity(pack.getShippingQuantity());
		details.setShippingWeight(pack.getBoxWeight());
		details.setShippingWidth(pack.getBoxWidth());
		details.setTreshold(pack.getTreshold());
		details.setType(pack.getShipPackageType().name());
		return details;
	}
	
	private com.salesmanager.core.model.shipping.Package toPackage(PackageDetails pack) {
		com.salesmanager.core.model.shipping.Package details = new com.salesmanager.core.model.shipping.Package();
		details.setCode(pack.getCode());
		details.setBoxHeight(pack.getShippingHeight());
		details.setBoxLength(pack.getShippingLength());
		details.setMaxWeight(pack.getShippingMaxWeight());
		//details.setShippingQuantity(pack.getShippingQuantity());
		details.setBoxWeight(pack.getShippingWeight());
		details.setBoxWidth(pack.getShippingWidth());
		details.setTreshold(pack.getTreshold());
		details.setShipPackageType(ShippingPackageType.valueOf(pack.getType()));
		return details;
	}

	@Override
	public List<ReadableCountry> shipToCountry(MerchantStore store, Language language) {
		
		
		try {
			List<Country> countries  = shippingService.getShipToCountryList(store, language);
			
			List<ReadableCountry> countryList = new ArrayList<ReadableCountry>();

			if(!CollectionUtils.isEmpty(countries)) {
				
				countryList = countries.stream()
				        .map(c -> {
							try {
								return convert(c, store, language);
							} catch (ConversionException e) {
								throw new ConversionRuntimeException("Error converting Country to readable country,e");
							}
						})
						.sorted(Comparator.comparing(ReadableCountry::getName))
						.collect(Collectors.toList());

			}
			
			return countryList;
		} catch (Exception e) {
			throw new ServiceRuntimeException("Error getting shipping country", e);
		}

		
		
		
	}
	
	ReadableCountry convert(Country country, MerchantStore store, Language lang) throws ConversionException {
		ReadableCountryPopulator countryPopulator = new ReadableCountryPopulator();
		return countryPopulator.populate(country, store, lang);
	}

}



```
