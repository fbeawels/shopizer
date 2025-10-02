# StoreFacadeImpl.java

## Review

## 1. Summary  
**Purpose** – `StoreFacadeImpl` is a Spring‑managed service that exposes CRUD and query operations for merchant stores and their associated data (logo, brand, configuration, languages). It sits on top of a number of business services (`MerchantStoreService`, `MerchantConfigurationService`, `LanguageService`, `ContentService`) and translates between *business* entities (`MerchantStore`, `MerchantConfiguration`) and *API* DTOs (`ReadableMerchantStore`, `ReadableBrand`, `PersistableMerchantStore`, etc.).

**Key components**

| Component | Role |
|-----------|------|
| `MerchantStoreService` | Persist/retrieve `MerchantStore` entities |
| `MerchantConfigurationService` | Persist/retrieve store‑level configuration |
| `LanguageService` | Resolve language information |
| `ContentService` | Store and delete files (e.g. logo images) |
| `PersistableMerchantStorePopulator` / `ReadableMerchantStorePopulator` | Map between entity and DTO |
| `ImageFilePath` | Build file‑system paths for images |

The facade largely delegates to these services, performing only very light orchestration (e.g. validation, exception wrapping, simple business‑rule checks such as “don’t delete the default store”).

---

## 2. Detailed Description  

### Flow of execution  

| Step | Method | What it does |
|------|--------|--------------|
| 1. **Lookup** | `getByCode(HttpServletRequest)` | Extracts the `store` request parameter, falls back to the default store code and forwards to `get(String)` |
| 2. **Retrieve** | `get(String)` | Calls `merchantStoreService.getByCode(code)` and wraps any `ServiceException` into a `ServiceRuntimeException` |
| 3. **DTO conversion** | `getByCode(String, Language)` / `getFullByCode(String, Language)` | Retrieves the `MerchantStore` and converts it to a `ReadableMerchantStore` using the populator |
| 4. **Create/Update** | `create(...)` / `update(...)` | Validates the incoming DTO, converts it to an entity via the populator, then delegates to `merchantStoreService` |
| 5. **Search** | `getByCriteria(...)` / `findAll(...)` | Builds criteria, invokes the underlying service, maps the results to `ReadableMerchantStoreList` |
| 6. **Brand/Logo** | `getBrand(...)`, `addStoreLogo(...)`, `deleteLogo(...)` | Handles image file creation/deletion and retrieves brand‑related configuration |
| 7. **Misc.** | `supportedLanguages(...)`, `getChildStores(...)`, `getMerchantStoreNames(...)` | Various helper operations around languages, children and store names |

### Assumptions & Constraints  

* All services are expected to be correctly injected; the facade will throw `ServiceRuntimeException` if any underlying call fails.  
* The default store code is a constant (`Constants.DEFAULT_STORE`).  
* A store cannot be deleted if its code equals the default store (case‑insensitive).  
* When updating or creating a store, the DTO must contain a non‑null `code`.  
* The code expects the `MerchantStore` entity to be fully populated by the populator; any mapping error results in a `ConversionRuntimeException`.  

### Design Choices  

* **Separation of concerns** – Business logic is in services, DTO mapping is delegated to populators, and the facade only orchestrates and does light validation.  
* **Exception translation** – All checked `ServiceException` and `ConversionException` are converted to unchecked runtime exceptions for API consumers.  
* **Use of Spring’s `@Service` and dependency injection** – Keeps the class testable.  
* **Legacy code style** – Uses Apache Commons utilities, manual null checks, and explicit `try/catch` blocks instead of modern Java features (streams, `Optional` is used in a few places but not consistently).  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `getByCode(HttpServletRequest)` | Retrieve store based on request param | `HttpServletRequest` | `MerchantStore` | None |
| `get(String)` | Internal wrapper that converts `ServiceException` | `String code` | `MerchantStore` | Throws `ServiceRuntimeException` |
| `getByCode(String, String)` | Retrieve readable store for a language code | `String code, String lang` | `ReadableMerchantStore` | None |
| `getFullByCode(String, String)` | Same as above but for full details | `String code, String lang` | `ReadableMerchantStore` | None |
| `getByCode(String, Language)` | Retrieve readable store for a `Language` object | `String code, Language lang` | `ReadableMerchantStore` | None |
| `getFullByCode(String, Language)` | Same as above but full details | `String code, Language lang` | `ReadableMerchantStore` | None |
| `existByCode(String)` | Check existence | `String code` | `boolean` | None |
| `create(PersistableMerchantStore)` | Persist a new store | `PersistableMerchantStore` | `void` | Calls `merchantStoreService.saveOrUpdate` |
| `update(PersistableMerchantStore)` | Update an existing store | `PersistableMerchantStore` | `void` | Calls `merchantStoreService.update` |
| `getByCriteria(MerchantStoreCriteria, Language)` | Search stores by criteria | `MerchantStoreCriteria`, `Language` | `ReadableMerchantStoreList` | None |
| `delete(String)` | Delete a store | `String code` | `void` | Calls `merchantStoreService.delete` |
| `getBrand(String)` | Retrieve brand information (logo + social links) | `String code` | `ReadableBrand` | None |
| `deleteLogo(String)` | Remove a store logo | `String code` | `void` | Calls `merchantStoreService` & `contentService` |
| `addStoreLogo(String, InputContentFile)` | Set and persist store logo | `String code, InputContentFile` | `void` | Calls `contentService.addLogo` |
| `createBrand(String, PersistableBrand)` | Persist brand social‑network configs | `String code, PersistableBrand` | `void` | Calls `merchantConfigurationService` |
| `getChildStores(Language, String, int, int)` | List child stores (retailer only) | `Language, String code, int page, int count` | `ReadableMerchantStoreList` | None |
| `findAll(MerchantStoreCriteria, Language, int, int)` | Find all stores matching criteria with paging | `MerchantStoreCriteria, Language, int, int` | `ReadableMerchantStoreList` | None |
| `getMerchantStoreNames(MerchantStoreCriteria)` | Retrieve a list of store names | `MerchantStoreCriteria` | `List<ReadableMerchantStore>` | None |
| `supportedLanguages(MerchantStore)` | Return supported languages for a store | `MerchantStore` | `List<Language>` | None |
| *Helper methods* (`convertMerchantStoreToReadableMerchantStore*`, `convertPersistableMerchantStoreToMerchantStore`, etc.) | Map between entities and DTOs | Various | Various | None |

---

## 4. Dependencies  

| Library / Framework | Type | Purpose |
|---------------------|------|---------|
| **Spring Framework** (`@Service`, `@Inject`, `@Autowired`) | 3rd‑party | Dependency injection and service declaration |
| **Spring Data** (`Page`) | 3rd‑party | Paging support |
| **Apache Commons Collections4** (`CollectionUtils`) | 3rd‑party | Collection utilities |
| **Apache Commons Lang3** (`Validate`) | 3rd‑party | Validation utilities |
| **Apache Commons Lang3** (`StringUtils`) | 3rd‑party (actually `org.drools.core.util.StringUtils` used here) | String utilities |
| **SLF4J** (`Logger`, `LoggerFactory`) | 3rd‑party | Logging |
| **SalesManager core / shop** (e.g. `MerchantStoreService`, `MerchantConfigurationService`, `PersistableMerchantStorePopulator`, etc.) | Internal | Business logic & data models |
| **Java Servlet API** (`HttpServletRequest`) | Standard | Request handling for `getByCode(HttpServletRequest)` |

---

## 5. Additional Notes & Suggested Improvements  

### 5.1 Code‑Quality / Maintainability  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Duplicate conversion methods** (`convertMerchantStoreToReadableMerchantStore` vs `convertMerchantStoreToReadableMerchantStoreWithFullDetails`) | Increases code size and risk of divergence | Merge into a single method; if future differences exist, add a flag or separate method that calls the common logic. |
| **Method name clash** (`getByCode(String)` defined twice – one internal, one public) | Compiles, but confusing for developers | Rename internal `get(String)` to something like `getMerchantStoreByCodeInternal(String)` or make it `private`. |
| **Inconsistent exception handling** | Hard to trace failures | Adopt a single helper method that translates checked exceptions to runtime (`wrapServiceException`) or use Spring’s `@Transactional` with proper propagation. |
| **Manual null checks** (`StringUtils.isEmpty`, `Validate.notNull`) | Boilerplate code | Use Java 8 `Optional` or assert statements; consider Lombok’s `@NonNull` for constructor validation. |
| **Hard‑coded strings** (`MerchantStore.DEFAULT_STORE`) | Magic values | Move to a constant in a dedicated class (e.g., `StoreConstants.DEFAULT_STORE_CODE`). |
| **Unnecessary casts** (`(List<ReadableMerchantStore>) stores.getList().stream()...`) | Potential ClassCastException if stream type mismatches | Let the compiler infer the type: `stores.getList().stream().map(...).collect(Collectors.toList());` |
| **Use of `org.drools.core.util.StringUtils`** | Unrelated dependency | Replace with Apache Commons `org.apache.commons.lang3.StringUtils`. |
| **`getChildStores` uses `children.getSize()` for `recordsFiltered`** | Should be `children.getNumberOfElements()` or `children.getSize()`? (size = page size, not filtered) | Clarify intent and use correct paging metrics. |
| **`supportedLanguages` validates `store.getClass()`** | Redundant check | Remove; class will never be null. |
| **Logging** – messages lack context (e.g., `Error while deleting MerchantStore`). | Hard to debug multi‑tenant logs | Include store code in log message (`"Error while deleting MerchantStore [{}]".formatted(code)`). |
| **Mix of `@Inject` and `@Autowired`** | Inconsistent style | Use one injection style consistently (prefer Spring’s `@Autowired` or constructor injection). |
| **No unit tests** | Unverified behavior | Add unit tests for each CRUD path, especially error cases. |

### 5.2 Performance  

* The `findAll` and `getByCriteria` methods fetch all results into memory (stream mapping) before pagination; if the underlying service already paginates, this is fine.  
* `getBrand` loads all configuration for a store and then filters only social ones. If many configurations exist, consider retrieving only the needed type from the DAO.  

### 5.3 Security  

* `getByCode(HttpServletRequest)` exposes the store code directly from a request parameter. If this endpoint is publicly reachable, consider validating the caller’s permissions or sanitizing the input.  

### 5.4 Extensibility  

* **Brand configuration**: Currently only social networks are handled. If new configuration types (e.g., shipping, payment) are added, the `getBrand` logic would need to be extended. Abstract the configuration handling into a separate service.  
* **DTO mapping**: If more entities need to be exposed, the populator pattern should be expanded or replaced with a mapping framework (MapStruct, ModelMapper) to reduce boilerplate.

### 5.5 Documentation  

* Add Javadoc for public methods, especially those exposed via API.  
* Clarify the difference (if any) between “full” and “basic” store retrieval.

---

### Bottom‑Line

`StoreFacadeImpl` is a functional, well‑structured service that correctly delegates to underlying business services. However, the codebase contains several duplication, naming inconsistencies, and outdated patterns that can hinder maintenance and future extension. Refactoring the conversion logic, unifying exception handling, modernizing null checks, and cleaning up the injection style will significantly improve readability and robustness. Adding comprehensive unit tests and Javadoc will further strengthen the code quality.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.store.facade;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.Validate;
import org.drools.core.util.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ConversionException;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.content.ContentService;
import com.salesmanager.core.business.services.merchant.MerchantStoreService;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.business.services.system.MerchantConfigurationService;
import com.salesmanager.core.constants.MeasureUnit;
import com.salesmanager.core.model.common.GenericEntityList;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.merchant.MerchantStoreCriteria;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.system.MerchantConfiguration;
import com.salesmanager.core.model.system.MerchantConfigurationType;
import com.salesmanager.shop.model.content.ReadableImage;
import com.salesmanager.shop.model.store.MerchantConfigEntity;
import com.salesmanager.shop.model.store.PersistableBrand;
import com.salesmanager.shop.model.store.PersistableMerchantStore;
import com.salesmanager.shop.model.store.ReadableBrand;
import com.salesmanager.shop.model.store.ReadableMerchantStore;
import com.salesmanager.shop.model.store.ReadableMerchantStoreList;
import com.salesmanager.shop.populator.store.PersistableMerchantStorePopulator;
import com.salesmanager.shop.populator.store.ReadableMerchantStorePopulator;
import com.salesmanager.shop.store.api.exception.ConversionRuntimeException;
import com.salesmanager.shop.store.api.exception.ResourceNotFoundException;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;
import com.salesmanager.shop.utils.ImageFilePath;
import com.salesmanager.shop.utils.LanguageUtils;

@Service("storeFacade")
public class StoreFacadeImpl implements StoreFacade {

	@Inject
	private MerchantStoreService merchantStoreService;

	@Inject
	private MerchantConfigurationService merchantConfigurationService;

	@Inject
	private LanguageService languageService;

	@Inject
	private ContentService contentService;

	@Inject
	private PersistableMerchantStorePopulator persistableMerchantStorePopulator;

	@Inject
	@Qualifier("img")
	private ImageFilePath imageUtils;

	@Inject
	private LanguageUtils languageUtils;
	
	@Autowired
	private ReadableMerchantStorePopulator readableMerchantStorePopulator;

	private static final Logger LOG = LoggerFactory.getLogger(StoreFacadeImpl.class);

	@Override
	public MerchantStore getByCode(HttpServletRequest request) {
		String code = request.getParameter("store");
		if (StringUtils.isEmpty(code)) {
			code = com.salesmanager.core.business.constants.Constants.DEFAULT_STORE;
		}
		return get(code);
	}

	@Override
	public MerchantStore get(String code) {
		try {
			MerchantStore store = merchantStoreService.getByCode(code);
			return store;
		} catch (ServiceException e) {
			LOG.error("Error while getting MerchantStore", e);
			throw new ServiceRuntimeException(e);
		}

	}

	@Override
	public ReadableMerchantStore getByCode(String code, String lang) {
		Language language = getLanguage(lang);
		return getByCode(code, language);
	}

	@Override
	public ReadableMerchantStore getFullByCode(String code, String lang) {
		Language language = getLanguage(lang);
		return getFullByCode(code, language);
	}

	private Language getLanguage(String lang) {
		return languageUtils.getServiceLanguage(lang);
	}

	@Override
	public ReadableMerchantStore getByCode(String code, Language language) {
		MerchantStore store = getMerchantStoreByCode(code);
		return convertMerchantStoreToReadableMerchantStore(language, store);
	}

	@Override
	public ReadableMerchantStore getFullByCode(String code, Language language) {
		MerchantStore store = getMerchantStoreByCode(code);
		return convertMerchantStoreToReadableMerchantStoreWithFullDetails(language, store);
	}

	@Override
	public boolean existByCode(String code) {
		try {
			return merchantStoreService.getByCode(code) != null;
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}
	}

	private ReadableMerchantStore convertMerchantStoreToReadableMerchantStore(Language language, MerchantStore store) {
		ReadableMerchantStore readable = new ReadableMerchantStore();

		/**
		 * Language is not important for this conversion using default language
		 */
		try {			
			readableMerchantStorePopulator.populate(store, readable, store, language);
		} catch (Exception e) {
			throw new ConversionRuntimeException("Error while populating MerchantStore " + e.getMessage());
		}
		return readable;
	}

	private ReadableMerchantStore convertMerchantStoreToReadableMerchantStoreWithFullDetails(Language language, MerchantStore store) {
		ReadableMerchantStore readable = new ReadableMerchantStore();


		/**
		 * Language is not important for this conversion using default language
		 */
		try {
			readableMerchantStorePopulator.populate(store, readable, store, language);
		} catch (Exception e) {
			throw new ConversionRuntimeException("Error while populating MerchantStore " + e.getMessage());
		}
		return readable;
	}

	private MerchantStore getMerchantStoreByCode(String code) {
		return Optional.ofNullable(get(code))
				.orElseThrow(() -> new ResourceNotFoundException("Merchant store code [" + code + "] not found"));
	}

	@Override
	public void create(PersistableMerchantStore store) {

		Validate.notNull(store, "PersistableMerchantStore must not be null");
		Validate.notNull(store.getCode(), "PersistableMerchantStore.code must not be null");

		// check if store code exists
		MerchantStore storeForCheck = get(store.getCode());
		if (storeForCheck != null) {
			throw new ServiceRuntimeException("MerhantStore " + store.getCode() + " already exists");
		}

		MerchantStore mStore = convertPersistableMerchantStoreToMerchantStore(store, languageService.defaultLanguage());
		createMerchantStore(mStore);

	}

	private void createMerchantStore(MerchantStore mStore) {
		try {
			merchantStoreService.saveOrUpdate(mStore);
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}
	}

	private MerchantStore convertPersistableMerchantStoreToMerchantStore(PersistableMerchantStore store,
			Language language) {
		MerchantStore mStore = new MerchantStore();

		// set default values
		mStore.setWeightunitcode(MeasureUnit.KG.name());
		mStore.setSeizeunitcode(MeasureUnit.IN.name());

		try {
			mStore = persistableMerchantStorePopulator.populate(store, mStore, language);
		} catch (ConversionException e) {
			throw new ConversionRuntimeException(e);
		}
		return mStore;
	}

	@Override
	public void update(PersistableMerchantStore store) {

		Validate.notNull(store);

		MerchantStore mStore = mergePersistableMerchantStoreToMerchantStore(store, store.getCode(),
				languageService.defaultLanguage());

		updateMerchantStore(mStore);

	}

	private void updateMerchantStore(MerchantStore mStore) {
		try {
			merchantStoreService.update(mStore);
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}

	}

	private MerchantStore mergePersistableMerchantStoreToMerchantStore(PersistableMerchantStore store, String code,
			Language language) {

		MerchantStore mStore = getMerchantStoreByCode(code);

		store.setId(mStore.getId());

		try {
			mStore = persistableMerchantStorePopulator.populate(store, mStore, language);
		} catch (ConversionException e) {
			throw new ConversionRuntimeException(e);
		}
		return mStore;
	}

	@Override
	public ReadableMerchantStoreList getByCriteria(MerchantStoreCriteria criteria, Language lang) {
		return  getMerchantStoresByCriteria(criteria, lang);

	}



	private ReadableMerchantStoreList getMerchantStoresByCriteria(MerchantStoreCriteria criteria, Language language) {
		try {
			GenericEntityList<MerchantStore> stores =  Optional.ofNullable(merchantStoreService.getByCriteria(criteria))
					.orElseThrow(() -> new ResourceNotFoundException("Criteria did not match any store"));
			
			
			ReadableMerchantStoreList storeList = new ReadableMerchantStoreList();
			storeList.setData(
					(List<ReadableMerchantStore>) stores.getList().stream()
					.map(s -> convertMerchantStoreToReadableMerchantStore(language, s))
			        .collect(Collectors.toList())
					);
			storeList.setTotalPages(stores.getTotalPages());
			storeList.setRecordsTotal(stores.getTotalCount());
			storeList.setNumber(stores.getList().size());
			
			return storeList;
			
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}

	}

	@Override
	public void delete(String code) {

		if (MerchantStore.DEFAULT_STORE.equals(code.toUpperCase())) {
			throw new ServiceRuntimeException("Cannot remove default store");
		}

		MerchantStore mStore = getMerchantStoreByCode(code);

		try {
			merchantStoreService.delete(mStore);
		} catch (Exception e) {
			LOG.error("Error while deleting MerchantStore", e);
			throw new ServiceRuntimeException("Error while deleting MerchantStore " + e.getMessage());
		}

	}

	@Override
	public ReadableBrand getBrand(String code) {
		MerchantStore mStore = getMerchantStoreByCode(code);

		ReadableBrand readableBrand = new ReadableBrand();
		if (!StringUtils.isEmpty(mStore.getStoreLogo())) {
			String imagePath = imageUtils.buildStoreLogoFilePath(mStore);
			ReadableImage image = createReadableImage(mStore.getStoreLogo(), imagePath);
			readableBrand.setLogo(image);
		}
		List<MerchantConfigEntity> merchantConfigTOs = getMerchantConfigEntities(mStore);
		readableBrand.getSocialNetworks().addAll(merchantConfigTOs);
		return readableBrand;
	}

	private List<MerchantConfigEntity> getMerchantConfigEntities(MerchantStore mStore) {
		List<MerchantConfiguration> configurations = getMergeConfigurationsByStore(MerchantConfigurationType.SOCIAL,
				mStore);

		return configurations.stream().map(config -> convertToMerchantConfigEntity(config))
				.collect(Collectors.toList());
	}

	private List<MerchantConfiguration> getMergeConfigurationsByStore(MerchantConfigurationType configurationType,
			MerchantStore mStore) {
		try {
			return merchantConfigurationService.listByType(configurationType, mStore);
		} catch (ServiceException e) {
			throw new ServiceRuntimeException("Error wile getting merchantConfigurations " + e.getMessage());
		}
	}

	private MerchantConfigEntity convertToMerchantConfigEntity(MerchantConfiguration config) {
		MerchantConfigEntity configTO = new MerchantConfigEntity();
		configTO.setId(config.getId());
		configTO.setKey(config.getKey());
		configTO.setType(config.getMerchantConfigurationType());
		configTO.setValue(config.getValue());
		configTO.setActive(config.getActive() != null ? config.getActive().booleanValue() : false);
		return configTO;
	}

	private MerchantConfiguration convertToMerchantConfiguration(MerchantConfigEntity config,
			MerchantConfigurationType configurationType) {
		MerchantConfiguration configTO = new MerchantConfiguration();
		configTO.setId(config.getId());
		configTO.setKey(config.getKey());
		configTO.setMerchantConfigurationType(configurationType);
		configTO.setValue(config.getValue());
		configTO.setActive(new Boolean(config.isActive()));
		return configTO;
	}

	private ReadableImage createReadableImage(String storeLogo, String imagePath) {
		ReadableImage image = new ReadableImage();
		image.setName(storeLogo);
		image.setPath(imagePath);
		return image;
	}

	@Override
	public void deleteLogo(String code) {
		MerchantStore store = getByCode(code);
		String image = store.getStoreLogo();
		store.setStoreLogo(null);

		try {
			updateMerchantStore(store);
			if (!StringUtils.isEmpty(image)) {
				contentService.removeFile(store.getCode(), image);
			}
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e.getMessage());
		}
	}

	@Override
	public MerchantStore getByCode(String code) {
		return getMerchantStoreByCode(code);
	}

	@Override
	public void addStoreLogo(String code, InputContentFile cmsContentImage) {
		MerchantStore store = getByCode(code);
		store.setStoreLogo(cmsContentImage.getFileName());
		saveMerchantStore(store);
		addLogoToStore(code, cmsContentImage);
	}

	private void addLogoToStore(String code, InputContentFile cmsContentImage) {
		try {
			contentService.addLogo(code, cmsContentImage);
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}
	}

	private void saveMerchantStore(MerchantStore store) {
		try {
			merchantStoreService.save(store);
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}

	}

	@Override
	public void createBrand(String merchantStoreCode, PersistableBrand brand) {
		MerchantStore mStore = getMerchantStoreByCode(merchantStoreCode);

		List<MerchantConfigEntity> createdConfigs = brand.getSocialNetworks();

		List<MerchantConfiguration> configurations = createdConfigs.stream()
				.map(config -> convertToMerchantConfiguration(config, MerchantConfigurationType.SOCIAL))
				.collect(Collectors.toList());
		try {
			for (MerchantConfiguration mConfigs : configurations) {
				mConfigs.setMerchantStore(mStore);
				if (!StringUtils.isEmpty(mConfigs.getValue())) {
					mConfigs.setMerchantConfigurationType(MerchantConfigurationType.SOCIAL);
					merchantConfigurationService.saveOrUpdate(mConfigs);
				} else {// remove if submited blank and exists
					MerchantConfiguration config = merchantConfigurationService
							.getMerchantConfiguration(mConfigs.getKey(), mStore);
					if (config != null) {
						merchantConfigurationService.delete(config);
					}
				}
			}
		} catch (ServiceException se) {
			throw new ServiceRuntimeException(se);
		}

	}

	@Override
	public ReadableMerchantStoreList getChildStores(Language language, String code, int page, int count) {
		try {

			// first check if store is retailer
			MerchantStore retailer = this.getByCode(code);
			if (retailer == null) {
				throw new ResourceNotFoundException("Merchant [" + code + "] not found");
			}

			if (retailer.isRetailer() == null || !retailer.isRetailer().booleanValue()) {
				throw new ResourceNotFoundException("Merchant [" + code + "] not a retailer");
			}

			
			Page<MerchantStore> children = merchantStoreService.listChildren(code, page, count);
			List<ReadableMerchantStore> readableStores = new ArrayList<ReadableMerchantStore>();
			ReadableMerchantStoreList readableList = new ReadableMerchantStoreList();
			if (!CollectionUtils.isEmpty(children.getContent())) {
				for (MerchantStore store : children)
					readableStores.add(convertMerchantStoreToReadableMerchantStore(language, store));
			}
			readableList.setData(readableStores);
			readableList.setRecordsFiltered(children.getSize());
			readableList.setTotalPages(children.getTotalPages());
			readableList.setRecordsTotal(children.getTotalElements());
			readableList.setNumber(children.getNumber());
			
			return readableList;
			
			
			
/*			List<MerchantStore> children = merchantStoreService.listChildren(code);
			List<ReadableMerchantStore> readableStores = new ArrayList<ReadableMerchantStore>();
			if (!CollectionUtils.isEmpty(children)) {
				for (MerchantStore store : children)
					readableStores.add(convertMerchantStoreToReadableMerchantStore(language, store));
			}
			return readableStores;*/
		} catch (ServiceException e) {
			throw new ServiceRuntimeException(e);
		}

	}

	@Override
	public ReadableMerchantStoreList findAll(MerchantStoreCriteria criteria, Language language, int page, int count) {
		
		try {
			Page<MerchantStore> stores = null;
			List<ReadableMerchantStore> readableStores = new ArrayList<ReadableMerchantStore>();
			ReadableMerchantStoreList readableList = new ReadableMerchantStoreList();
			
			Optional<String> code = Optional.ofNullable(criteria.getStoreCode());
			Optional<String> name = Optional.ofNullable(criteria.getName());
			if(code.isPresent()) {
				
				stores = merchantStoreService.listByGroup(name, code.get(), page, count);

			} else {
				if(criteria.isRetailers()) {
					stores = merchantStoreService.listAllRetailers(name, page, count);
				} else {
					stores = merchantStoreService.listAll(name, page, count);
				}
			}


			if (!CollectionUtils.isEmpty(stores.getContent())) {
				for (MerchantStore store : stores)
					readableStores.add(convertMerchantStoreToReadableMerchantStore(language, store));
			}
			readableList.setData(readableStores);
			readableList.setRecordsTotal(stores.getTotalElements());
			readableList.setTotalPages(stores.getTotalPages());
			readableList.setNumber(stores.getSize());
			readableList.setRecordsFiltered(stores.getSize());
						return readableList;

		} catch (ServiceException e) {
			throw new ServiceRuntimeException("Error while finding all merchant", e);
		}


	}
	
	private ReadableMerchantStore convertStoreName(MerchantStore store) {
		ReadableMerchantStore convert = new ReadableMerchantStore();
		convert.setId(store.getId());
		convert.setCode(store.getCode());
		convert.setName(store.getStorename());
		return convert;
	}

	@Override
	public List<ReadableMerchantStore> getMerchantStoreNames(MerchantStoreCriteria criteria) {
		Validate.notNull(criteria, "MerchantStoreCriteria must not be null");
		
		try {
			
			List<ReadableMerchantStore> stores = null;
			Optional<String> code = Optional.ofNullable(criteria.getStoreCode());
			
			
			//TODO Pageable
			if(code.isPresent()) {
				
				stores = merchantStoreService.findAllStoreNames(code.get()).stream()
						.map(s -> convertStoreName(s))
						.collect(Collectors.toList());
			} else {
				stores = merchantStoreService.findAllStoreNames().stream()
						.map(s -> convertStoreName(s))
						.collect(Collectors.toList());
			}
			
			
			return stores;
		} catch (ServiceException e) {
			throw new ServiceRuntimeException("Exception while getting store name",e);
		}
		

	}

	@Override
	public List<Language> supportedLanguages(MerchantStore store) {
		
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notNull(store.getClass(), "MerchantStore code cannot be null");
		
		if(!CollectionUtils.isEmpty(store.getLanguages())) {
			return store.getLanguages();
		}
		
		//refresh
		try {
			store = merchantStoreService.getByCode(store.getCode());
		} catch (ServiceException e) {
			throw new ServiceRuntimeException("An exception occured when getting store [" + store.getCode() + "]");
		}
		
		if(store!=null) {
			return store.getLanguages();
		}
		
		return Collections.emptyList();
	}

}


```
