# LanguageUtils.java

## Review

## 1. Summary  

**Purpose**  
`LanguageUtils` is a Spring‐managed component that centralises the logic for resolving a language/locale for both MVC requests and REST endpoints. It bridges the gap between the HTTP request, the current session, the merchant store configuration and the underlying `LanguageService`/`StoreFacade`.  

**Key Components**  
| Component | Role |
|-----------|------|
| `LanguageService` | DAO/service that knows how to look up languages, convert to/from `Locale` and provide a default. |
| `StoreFacade` | Thin wrapper that fetches a `MerchantStore` (used for store‑specific default language). |
| `LocaleResolver` / `LocaleContextHolder` | Spring helpers that keep the current locale per request thread. |
| Constants (`Constants.LANGUAGE`, `Constants.MERCHANT_STORE`, `Constants.LANG`, `DEFAULT_STORE`) | Keys used for session attributes and query parameters. |

**Design Patterns / Libraries**  
* **Spring Dependency Injection** – `@Component`, `@Inject` / `@Autowired`.  
* **Facade** – `StoreFacade` abstracts the underlying store retrieval.  
* **Factory / Service** – `LanguageService` encapsulates language data access.  
* **Utility / Helper** – The class itself is a stateless helper that can be injected wherever language resolution is needed.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

| Method | Typical Execution Path | Key Points |
|--------|------------------------|------------|
| `getServiceLanguage(String)` | *Lookup* – tries to find a language by code, falls back to the default. | Straight‑forward, only catches `ServiceException`. |
| `getRequestLanguage(HttpServletRequest, HttpServletResponse)` | *MVC request* – <br>1. Try to read language from session. <br>2. If missing, use the browser locale (`LocaleContextHolder.getLocale()`), override with store default if a store is present. <br>3. Store the resolved language back into the session. <br>4. Set the locale on the `LocaleResolver` and the `HttpServletResponse`. | Handles missing store, missing language, and errors in language lookup. |
| `getRESTLanguage(HttpServletRequest, NativeWebRequest)` | *REST request* – <br>1. Pull `lang` query parameter. <br>2. If absent, try to determine the store from the `store` query parameter (or `DEFAULT_STORE`). <br>3. Use the store’s default language or the global default. <br>4. If a language code is supplied (and not “_all”) resolve it via `LanguageService`. | Returns the resolved language (or `null` when `lang=_all`). |

### 2.2 Assumptions & Constraints  

* The HTTP session is used to cache the `Language` and `MerchantStore` objects – assumes that each request belongs to a single user session.  
* The application uses a single `LocaleResolver` per web container, typically a `SessionLocaleResolver`.  
* `LanguageService` is assumed to be thread‑safe; it is injected once and reused.  
* For REST services, a request may include a `store` query parameter; if absent, the code falls back to a constant `DEFAULT_STORE`.  
* All language look‑ups are performed by code; if an unknown code is supplied, the global default is used.  

### 2.3 Architecture & Design Choices  

* **Stateless Utility** – `LanguageUtils` is effectively stateless apart from its injected services, making it thread‑safe.  
* **Explicit Session Caching** – Language and store are cached in the session, which keeps the request processing lightweight for repeated requests.  
* **Two Entry Points** – One for MVC (`getRequestLanguage`) and one for REST (`getRESTLanguage`), acknowledging that REST APIs often cannot rely on session state.  
* **Fallback Strategy** – The code follows a layered fallback: request param → session → browser locale → store default → global default.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getServiceLanguage(String lang)` | `Language getServiceLanguage(String)` | Resolve a language by its code; if not found return the system default. | `lang` – ISO‑639 language code | `Language` object (never `null`) | Logs an error if lookup fails. |
| `getRequestLanguage(HttpServletRequest request, HttpServletResponse response)` | `Language getRequestLanguage(HttpServletRequest, HttpServletResponse)` | Resolve language for a standard MVC request; sets session attribute and response locale. | `request`, `response` | `Language` resolved for the request | Stores `Language` and `MerchantStore` in session, sets locale on `LocaleResolver` and `HttpServletResponse`. |
| `getRESTLanguage(HttpServletRequest request, NativeWebRequest webRequest)` | `Language getRESTLanguage(HttpServletRequest, NativeWebRequest)` | Resolve language for REST APIs based on query parameters and store configuration. | `request`, `webRequest` | `Language` (may be `null` when `lang=_all`) | None beyond logging. |

### Reusable / Utility Methods  
* `languageService.toLocale(language, store)` – converts a `Language` to a `Locale`.  
* `languageService.toLanguage(Locale)` – reverse conversion.  
* `languageService.getByCode(String)` – DAO lookup.  

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| Spring Framework (`spring-core`, `spring-context`, `spring-web`) | Third‑party | Provides dependency injection, locale resolution, `NativeWebRequest`. |
| Apache Commons Lang (`commons-lang3`) | Third‑party | For `StringUtils`, `Validate`. |
| Apache Commons Logging (`commons-logging`) | Third‑party | For `Log`. |
| Custom business packages (`com.salesmanager.*`) | Internal | `LanguageService`, `StoreFacade`, `MerchantStore`, `Language`. |
| Servlet API | Platform | `HttpServletRequest`, `HttpServletResponse`. |

No platform‑specific assumptions beyond a standard Java EE / Spring web container.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Potential Issues  

1. **Null `store` in `getRequestLanguage`**  
   * If `store` is `null` but `language` is still `null`, the code falls back to the browser locale. However, later the call to `languageService.toLocale(language, store)` would pass `null` for both arguments if `language` remains `null`. While the service likely handles this, it is not explicitly documented.  

2. **`ALL_LANGUALES` Handling in REST**  
   * When the `lang` parameter equals `_all`, the method returns `null`. This is intentional but undocumented; callers must be prepared to handle a `null` language.  

3. **Redundant `StringUtils.isNotBlank` / `isBlank`**  
   * In `getRESTLanguage` the code filters the `store` parameter with `StringUtils::isNotBlank` and then immediately checks `!StringUtils.isBlank(storeValue)`. The second check is unnecessary and can be removed for clarity.  

4. **Exception Handling**  
   * `getRESTLanguage` only catches `ServiceException`. Any other exception (e.g., `NumberFormatException` if the store code is numeric) will propagate as an unchecked exception. It might be safer to catch `Exception` and wrap it in `ServiceRuntimeException`.  

5. **Locale Context vs. Browser Locale**  
   * The code obtains the browser locale via `LocaleContextHolder.getLocale()`. In a typical Spring MVC setup this reflects the locale set by the `LocaleResolver`. However, if no `LocaleResolver` has been configured, this may default to `Locale.getDefault()`. A more explicit call to `request.getLocale()` could be clearer.  

### 5.2 Potential Improvements  

| Area | Suggested Change |
|------|------------------|
| **Code Clarity** | Replace the nested `if`/`try/catch` blocks in `getRequestLanguage` with a small helper method that encapsulates the fallback chain. |
| **Logging** | Use SLF4J instead of Commons Logging for modernity and easier integration. |
| **Configuration** | Inject the default store code as a Spring `@Value("${store.default}")` instead of hard‑coding `DEFAULT_STORE`. |
| **Return Contracts** | Document that `getRESTLanguage` can return `null` when `lang=_all`. |
| **Thread‑Safety** | Explicitly declare the class `@Scope("singleton")` (default) to highlight the stateless nature. |
| **Testing** | Add unit tests that cover all fallback branches (e.g., missing language, missing store, invalid language code). |
| **Internationalisation** | Consider using `org.springframework.context.i18n.LocaleContextHolder.setLocale` only after confirming that `LocaleResolver` is present. |

### 5.3 Future Enhancements  

* **User‑Specific Language Preference** – Store a language preference in the user profile (e.g., in a database) and use that over session or store defaults.  
* **Header‑Based Language** – For REST APIs, support the `Accept-Language` header to auto‑detect the preferred language.  
* **Caching** – Cache language look‑ups in a `ConcurrentMap` for read‑heavy scenarios, especially if the underlying service is expensive.  
* **Configuration‑Driven Fallbacks** – Expose the fallback order via application properties to allow administrators to tweak it without code changes.  

---  

**Overall Assessment**  
`LanguageUtils` fulfills its role as a language/locale resolution helper. Its implementation is clear, but the code can be simplified for readability, made more robust against null values, and better documented regarding its contract with callers. The reliance on session attributes is appropriate for a web application but should be carefully considered if the application scales to a stateless microservice environment.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import static com.salesmanager.core.business.constants.Constants.DEFAULT_STORE;

import java.util.Locale;
import java.util.Optional;

import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.support.RequestContextUtils;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.constants.Constants;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;
import com.salesmanager.shop.store.controller.store.facade.StoreFacade;

@Component
public class LanguageUtils {

  protected final Log logger = LogFactory.getLog(getClass());
  public static final String REQUEST_PARAMATER_STORE = "store";

  private static final String ALL_LANGUALES = "_all";

  @Inject
  LanguageService languageService;
  
  @Autowired
  private StoreFacade storeFacade;

  public Language getServiceLanguage(String lang) {
    Language l = null;
    if (!StringUtils.isBlank(lang)) {
      try {
        l = languageService.getByCode(lang);
      } catch (ServiceException e) {
        logger.error("Cannot retrieve language " + lang, e);
      }
    }

    if (l == null) {
      l = languageService.defaultLanguage();
    }

    return l;
  }

  /**
   * Determines request language based on store rules
   * 
   * @param request
   * @return
   */
  public Language getRequestLanguage(HttpServletRequest request, HttpServletResponse response) {

    Locale locale = null;

    Language language = (Language) request.getSession().getAttribute(Constants.LANGUAGE);
    MerchantStore store =
        (MerchantStore) request.getSession().getAttribute(Constants.MERCHANT_STORE);
    


    if (language == null) {
      try {

        locale = LocaleContextHolder.getLocale();// should be browser locale



        if (store != null) {
          language = store.getDefaultLanguage();
          if (language != null) {
            locale = languageService.toLocale(language, store);
            if (locale != null) {
              LocaleContextHolder.setLocale(locale);
            }
            request.getSession().setAttribute(Constants.LANGUAGE, language);
          }

          if (language == null) {
            language = languageService.toLanguage(locale);
            request.getSession().setAttribute(Constants.LANGUAGE, language);
          }

        }

      } catch (Exception e) {
        if (language == null) {
          try {
            language = languageService.getByCode(Constants.DEFAULT_LANGUAGE);
          } catch (Exception ignore) {
          }
        }
      }
    } else {


      Locale localeFromContext = LocaleContextHolder.getLocale();// should be browser locale
      if (!language.getCode().equals(localeFromContext.getLanguage())) {
        // get locale context
        language = languageService.toLanguage(localeFromContext);
      }

    }

    if (language != null) {
      locale = languageService.toLocale(language, store);
    } else {
      language = languageService.toLanguage(locale);
    }

    LocaleResolver localeResolver = RequestContextUtils.getLocaleResolver(request);
    if (localeResolver != null) {
      localeResolver.setLocale(request, response, locale);
    }
    response.setLocale(locale);
    request.getSession().setAttribute(Constants.LANGUAGE, language);

    return language;
  }

  /**
   * Should be used by rest web services
   * 
   * @param request
   * @param store
   * @return
   * @throws Exception
   */
  public Language getRESTLanguage(HttpServletRequest request, NativeWebRequest webRequest) {

    Validate.notNull(request, "HttpServletRequest must not be null");

    try {
      Language language = null;

      String lang = request.getParameter(Constants.LANG);

      if (StringUtils.isBlank(lang)) {
        if (language == null) {
    	    String storeValue = Optional.ofNullable(webRequest.getParameter(REQUEST_PARAMATER_STORE))
    				.filter(StringUtils::isNotBlank).orElse(DEFAULT_STORE);
    	    if(!StringUtils.isBlank(storeValue)) {
    	      try {
    	    	  MerchantStore storeModel = storeFacade.get(storeValue);
    	    	  language = storeModel.getDefaultLanguage();
    	      } catch (Exception e) {
    	    	  logger.warn("Cannot get store with code [" + storeValue + "]");
    	      }
    	    	
    	    } else {
    	    	language = languageService.defaultLanguage();
    	    }

        }
      } else {
        if(!ALL_LANGUALES.equals(lang)) {
          language = languageService.getByCode(lang);
          if (language == null) {
            language = languageService.defaultLanguage();
          }
        }
      }
      
      //if language is null then underlying facade must load all languages
      return language;

    } catch (ServiceException e) {
      throw new ServiceRuntimeException(e);
    }
  }

}



```
