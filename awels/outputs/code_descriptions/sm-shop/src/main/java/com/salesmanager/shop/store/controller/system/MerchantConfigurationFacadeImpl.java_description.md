# MerchantConfigurationFacadeImpl.java

## Review

## 1. Summary  

**Purpose**  
`MerchantConfigurationFacadeImpl` is a Spring‑managed service that exposes a façade for retrieving merchant‑specific configuration data. It collects a mixture of boolean flags and optional social‑media URLs from the underlying `MerchantConfigurationService`, maps them into a `Configs` DTO, and performs a few lightweight conversions (e.g., a property‑driven Boolean flag).  

**Key components**  
| Component | Role |
|-----------|------|
| `MerchantConfigurationFacadeImpl` | Service layer façade; implements `MerchantConfigurationFacade`. |
| `MerchantConfigurationService` | Domain service that talks to the persistence layer. |
| `Configs` | DTO returned to the API layer (likely a REST controller). |
| `MerchantConfig` | Internal entity holding the bulk of boolean flags. |
| `MerchantConfiguration` | Entity holding a single key/value pair for optional settings. |
| Constants (`KEY_FACEBOOK_PAGE_URL`, …) | Centralised keys for optional config values. |
| `@Value("${config.displayShipping}")` | Spring property injection for a global shipping display flag. |

**Design patterns & frameworks**  
* **Facade** – Provides a simplified interface over the domain service.  
* **Optional** – Avoids explicit null checks when retrieving optional config values.  
* **Spring DI** – `@Service`, `@Inject`, `@Value` for dependency and property injection.  
* **SLF4J** – Logging.  

---

## 2. Detailed Description  

### Execution flow  
1. **Client call** – An external controller or component calls  
   `getMerchantConfig(MerchantStore, Language)`.  
2. **Merchant config retrieval** – The method delegates to a private helper  
   `getMerchantConfig(MerchantStore)` which in turn calls the injected  
   `merchantConfigurationService.getMerchantConfig(...)`.  
3. **DTO construction** – The retrieved `MerchantConfig` is used to populate a new `Configs` object.  
4. **Optional values** – For each social‑media key the helper `getConfigValue` pulls the value via  
   `merchantConfigurationService.getMerchantConfiguration(key, merchantStore)` and places it into the DTO if present.  
5. **Display‑shipping flag** – The string value from `@Value` is parsed to a Boolean and set on the DTO (defaulting to `false` if parsing fails or the property is missing).  
6. **Return** – The fully populated `Configs` object is returned to the caller.

### Assumptions & constraints  
* The underlying `MerchantConfigurationService` throws `ServiceException`.  
* `MerchantConfig` is never `null` (the code does not guard against this).  
* The `Language` argument is currently unused – it may be a legacy artifact.  
* The global `displayShipping` flag is optional; the code falls back to `false` if it is absent or malformed.  
* Thread safety is guaranteed by Spring’s singleton scope; the service holds no mutable state.  

### Architectural notes  
* **Layering** – A thin façade layer sits above a domain service.  
* **Error handling** – Domain exceptions are converted to unchecked `ServiceRuntimeException`.  
* **Optional usage** – Makes null handling explicit but could be simplified with helper methods or a builder pattern.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `public Configs getMerchantConfig(MerchantStore merchantStore, Language language)` | Public entry point; returns merchant‑specific configuration DTO. | `merchantStore` – target store; `language` – currently unused. | `Configs` – populated DTO. | None. |
| `private MerchantConfig getMerchantConfig(MerchantStore merchantStore)` | Retrieves `MerchantConfig` from the service; wraps checked exception. | `merchantStore` | `MerchantConfig` | Throws `ServiceRuntimeException` on `ServiceException`. |
| `private Optional<String> getConfigValue(String keyContant, MerchantStore merchantStore)` | Helper to fetch a single optional configuration value. | `keyContant` – configuration key; `merchantStore` | `Optional<String>` containing the value if present. | None. |
| `private Optional<MerchantConfiguration> getMerchantConfiguration(String key, MerchantStore merchantStore)` | Low‑level fetch of a `MerchantConfiguration` entity; converts checked exception. | `key`; `merchantStore` | `Optional<MerchantConfiguration>` | Throws `ServiceRuntimeException` on `ServiceException`. |

**Reusable/utility aspects**  
* The use of `Optional` and the mapping pattern (`.map(MerchantConfiguration::getValue)`) is a small but reusable idiom for optional field extraction.  
* The `@Value` injection pattern is standard and can be reused across the codebase.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.stereotype.Service` | Spring framework | Marks the class as a Spring bean. |
| `javax.inject.Inject` | JSR‑330 | Spring supports this annotation for DI. |
| `org.springframework.beans.factory.annotation.Value` | Spring framework | Injects a property value. |
| `org.apache.commons.lang3.StringUtils` | Third‑party (Apache Commons Lang) | Provides `isBlank` check. |
| `org.slf4j.Logger` / `LoggerFactory` | Third‑party (SLF4J) | Logging facade. |
| `com.salesmanager.core.business.services.system.MerchantConfigurationService` | Application | Domain service for configuration retrieval. |
| `com.salesmanager.core.model.*` | Application | Domain entities. |
| `com.salesmanager.shop.model.system.Configs` | Application | DTO. |
| `com.salesmanager.shop.store.api.exception.ServiceRuntimeException` | Application | Runtime wrapper for checked exceptions. |

All external libraries are standard (Spring, SLF4J, Apache Commons) and well‑supported. No platform‑specific APIs beyond Spring are used.

---

## 5. Additional Notes  

### 5.1 Unused parameters  
* The `Language language` argument in `getMerchantConfig(MerchantStore, Language)` is not used. If language‑specific logic is not required, the parameter should be removed to avoid confusion and reduce the method signature.

### 5.2 Null safety  
* `getMerchantConfig(MerchantStore)` assumes `merchantConfigurationService.getMerchantConfig()` never returns `null`. If a `null` can occur, the method should guard against it (e.g., return a default config or throw a clear exception).  
* The public `getMerchantConfig` method directly calls getters on the returned `MerchantConfig`. If the helper ever returns `null`, this will lead to an NPE. A defensive check or a `Objects.requireNonNull` call would improve robustness.

### 5.3 Logging & error handling  
* The catch block in `getMerchantConfig(MerchantStore)` silently rethrows a `ServiceRuntimeException` without logging the original exception. Adding a log entry would aid diagnostics.  
* When parsing the `displayShipping` property, the catch block logs a message but includes the raw string (`displayShipping`) without context. It could log the key name and the problematic value for clarity.

### 5.4 Property parsing  
* `Boolean.valueOf(displayShipping)` will silently return `false` for any non‑`"true"` value. The code already falls back to `false` in case of an exception, but the exception handling is redundant because `valueOf` never throws. The try‑catch block could be removed or replaced with a more explicit `Boolean.parseBoolean` if stricter parsing is desired.

### 5.5 Optional handling  
* The pattern `Optional.ofNullable(...).map(...)` is concise but somewhat verbose. If the project already has a utility method (e.g., `Optional.ofNullable(...).flatMap(Optional::ofNullable)`) this could be reused.  
* Consider using a small helper to combine key and store into a single method that returns the value directly, reducing repetition.

### 5.6 Future enhancements  
1. **Caching** – Configuration values rarely change; a simple LRU or time‑based cache could reduce service calls.  
2. **Validation** – Centralise validation of configuration keys and values (e.g., URL format) in a dedicated validator component.  
3. **Internationalisation** – If the `Language` argument becomes relevant, implement locale‑specific overrides or translation of certain config fields.  
4. **Testing** – Add unit tests covering the happy path, missing optional values, and exception scenarios.  
5. **API contract** – Expose `Configs` through a dedicated REST controller and document the expected JSON structure.  

---

**Overall**  
The implementation is concise, uses well‑known Spring conventions, and leverages `Optional` to avoid null checks. Minor refactoring (removing unused parameters, tightening null safety, improving logging, and simplifying property parsing) would make the code more robust and maintainable. The façade pattern is appropriate for the use case, and the code aligns well with the existing domain model.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.system;

import static com.salesmanager.shop.constants.Constants.KEY_FACEBOOK_PAGE_URL;
import static com.salesmanager.shop.constants.Constants.KEY_GOOGLE_ANALYTICS_URL;
import static com.salesmanager.shop.constants.Constants.KEY_INSTAGRAM_URL;
import static com.salesmanager.shop.constants.Constants.KEY_PINTEREST_PAGE_URL;

import java.util.Optional;

import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.system.MerchantConfigurationService;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.system.MerchantConfig;
import com.salesmanager.core.model.system.MerchantConfiguration;
import com.salesmanager.shop.model.system.Configs;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;


@Service
public class MerchantConfigurationFacadeImpl implements MerchantConfigurationFacade {

  private static final Logger LOGGER = LoggerFactory
      .getLogger(MerchantConfigurationFacadeImpl.class);

  @Inject
  private MerchantConfigurationService merchantConfigurationService;

  @Value("${config.displayShipping}")
  private String displayShipping;

  @Override
  public Configs getMerchantConfig(MerchantStore merchantStore, Language language) {

    MerchantConfig configs = getMerchantConfig(merchantStore);

    Configs readableConfig = new Configs();
    readableConfig.setAllowOnlinePurchase(configs.isAllowPurchaseItems());
    readableConfig.setDisplaySearchBox(configs.isDisplaySearchBox());
    readableConfig.setDisplayContactUs(configs.isDisplayContactUs());

    readableConfig.setDisplayCustomerSection(configs.isDisplayCustomerSection());
    readableConfig.setDisplayAddToCartOnFeaturedItems(configs.isDisplayAddToCartOnFeaturedItems());
    readableConfig.setDisplayCustomerAgreement(configs.isDisplayCustomerAgreement());
    readableConfig.setDisplayPagesMenu(configs.isDisplayPagesMenu());

    Optional<String> facebookConfigValue = getConfigValue(KEY_FACEBOOK_PAGE_URL, merchantStore);
    facebookConfigValue.ifPresent(readableConfig::setFacebook);

    Optional<String> googleConfigValue = getConfigValue(KEY_GOOGLE_ANALYTICS_URL, merchantStore);
    googleConfigValue.ifPresent(readableConfig::setGa);

    Optional<String> instagramConfigValue = getConfigValue(KEY_INSTAGRAM_URL, merchantStore);
    instagramConfigValue.ifPresent(readableConfig::setInstagram);


    Optional<String> pinterestConfigValue = getConfigValue(KEY_PINTEREST_PAGE_URL, merchantStore);
    pinterestConfigValue.ifPresent(readableConfig::setPinterest);

    readableConfig.setDisplayShipping(false);
    try {
      if(!StringUtils.isBlank(displayShipping)) {
        readableConfig.setDisplayShipping(Boolean.valueOf(displayShipping));
      }
    } catch(Exception e) {
      LOGGER.error("Cannot parse value of " + displayShipping);
    }

    return readableConfig;
  }

  private MerchantConfig getMerchantConfig(MerchantStore merchantStore) {
    try{
      return merchantConfigurationService.getMerchantConfig(merchantStore);
    } catch (ServiceException e){
      throw new ServiceRuntimeException(e);
    }
  }

  private Optional<String> getConfigValue(String keyContant, MerchantStore merchantStore) {
    return getMerchantConfiguration(keyContant, merchantStore)
        .map(MerchantConfiguration::getValue);
  }

  private Optional<MerchantConfiguration> getMerchantConfiguration(String key, MerchantStore merchantStore) {
    try{
      return Optional.ofNullable(merchantConfigurationService.getMerchantConfiguration(key, merchantStore));
    } catch (ServiceException e) {
      throw new ServiceRuntimeException(e);
    }

  }
}



```
