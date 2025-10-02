# EmailUtils.java

## Review

## 1. Summary  
**Purpose** – `EmailUtils` is a Spring‐managed component that prepares a key/value map of template tokens used to build generic HTML e‑mail messages.  
**Key responsibilities**  

| Component | Role |
|-----------|------|
| `EmailUtils` | Generates a `Map<String,String>` of placeholders that can be substituted into an e‑mail template (store name, logo URL, copyright, etc.). |
| `ImageFilePath` (`imageUtils`) | Helper that builds the relative file path to a store logo. |
| `LabelUtils` (`messages`) | Provides internationalised message strings. |
| `MerchantStore` (`store`) | Holds store‑specific data (name, email, domain, logo). |
| `Constants.HTTP_SCHEME` | Supplies the scheme (http/https) for constructing the logo URL. |

The class uses a simple **Factory** pattern: it does not create e‑mail objects itself but supplies the data needed by another component that actually renders the e‑mail.

---

## 2. Detailed Description  

### Flow of Execution  

1. **Bean creation** – Spring instantiates `EmailUtils` as a singleton (`@Component`) and injects an `ImageFilePath` implementation (`@Qualifier("img")`).
2. **Method call** – `createEmailObjectsMap()` is invoked with the HTTP context path, a `MerchantStore`, a `LabelUtils` instance, and a `Locale`.
3. **Token construction**  
   * Builds arrays of arguments used for localisation (`adminNameArg`, `adminEmailArg`, `copyArg`).  
   * Calls `messages.getMessage()` to obtain translated strings.  
   * Builds a logo URL when `store.getStoreLogo()` is present, otherwise falls back to the store name.  
   * Adds all tokens to a `HashMap` and returns it.  
4. **No cleanup** – The method is stateless; each call uses a fresh map and performs no resource deallocation.

### Design Choices  

* **Dependency Injection** – The class relies on Spring DI to obtain `ImageFilePath`. Mixing `@Inject` (JSR‑330) with Spring’s `@Qualifier` is acceptable but could be streamlined to Spring’s own `@Autowired`.  
* **Hard‑coded Scheme** – The URL for the logo is built with a constant scheme (`Constants.HTTP_SCHEME`). This presumes all stores use the same scheme; if a store is HTTPS while the constant is HTTP the resulting URL will be incorrect.  
* **Mutable Map** – A `HashMap` is returned directly; callers may inadvertently modify the map. An unmodifiable view could guard against accidental mutation.  
* **Error Handling** – The method assumes that none of its parameters are `null` and that `messages.getMessage()` never throws. There are no null‑checks or exception handling, which could lead to `NullPointerException`s at runtime.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `createEmailObjectsMap(String contextPath, MerchantStore store, LabelUtils messages, Locale locale)` | Builds a map of tokens for e‑mail templates. | *`contextPath`* – the web app context path.<br>*`store`* – merchant store information.<br>*`messages`* – localisation utility.<br>*`locale`* – target locale. | `Map<String,String>` – keys such as `EMAIL_STORE_NAME`, `LOGOPATH`, etc. | None. Creates a new `HashMap` and populates it. |

**Utility/Helper methods** – None exposed; all logic resides inside the single public method.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.beans.factory.annotation.Qualifier` | Spring core | Used to specify which `ImageFilePath` implementation to inject. |
| `javax.inject.Inject` | JSR‑330 | Provides constructor/setter injection; could be replaced with Spring’s `@Autowired`. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Application domain | Represents store configuration (name, logo, domain, email). |
| `com.salesmanager.shop.constants.Constants` | Application constants | Provides `HTTP_SCHEME`. |
| `com.salesmanager.shop.utils.ImageFilePath` | Application util | Builds relative logo path. |
| `com.salesmanager.shop.utils.LabelUtils` | Application util | Localises message keys. |
| `com.salesmanager.shop.utils.DateUtil` | Application util | Supplies current year. |

All dependencies are **third‑party or application‑specific**; there are no external HTTP clients, databases, or other heavy frameworks involved.

---

## 5. Additional Notes  

### Edge Cases & Missing Checks  

| Scenario | Issue | Recommendation |
|----------|-------|----------------|
| `store` is `null` | NPE when accessing `store.getStorename()` | Validate and throw a meaningful `IllegalArgumentException`. |
| `messages` is `null` | NPE on `messages.getMessage()` | Same as above. |
| `locale` is `null` | NPE inside `messages.getMessage()` | Provide default locale or validate. |
| `store.getStoreLogo()` is an empty string | Logo path may become malformed | Treat empty string the same as `null`. |
| `contextPath` is `null` | `logoPath` will contain `null` string | Provide default or empty string. |
| `Constants.HTTP_SCHEME` mis‑matches actual protocol | Broken image link | Retrieve scheme from the request or store configuration. |

### Potential Enhancements  

1. **Immutability** – Return `Collections.unmodifiableMap(templateTokens)` to prevent accidental mutation.  
2. **Locale Defaulting** – Use `Locale.getDefault()` if `locale` is `null`.  
3. **Scheme Handling** – Pass the scheme as a parameter or obtain it from the `MerchantStore`/`HttpServletRequest`.  
4. **Null‑Safe StringBuilder** – Use `String.format()` or a templating library (e.g., Thymeleaf) for building the HTML snippet.  
5. **Unit Tests** – Add tests covering null arguments, missing logo, different locales, and scheme variations.  
6. **Documentation** – Expand JavaDoc to describe each placeholder key and its intended usage.  
7. **DI Annotation Consistency** – Prefer `@Autowired` with `@Qualifier` to stay within Spring’s annotations, unless there is a reason to mix JSR‑330.  

### Security & Injection Concerns  

* The logo URL is constructed by concatenating strings; ensure that `store.getDomainName()` and the image path are sanitized to avoid injection attacks.  
* If any user‑supplied data ends up in the template (e.g., via `store.getStorename()`), consider escaping HTML.

---  

**Overall** – The component is straightforward and fits well into a Spring‑based mail‑generation pipeline. Minor robustness improvements (null checks, immutability, scheme handling) would make it more production‑ready.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

import javax.inject.Inject;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.shop.constants.Constants;


@Component
public class EmailUtils {
	
	private final static String EMAIL_STORE_NAME = "EMAIL_STORE_NAME";
	private final static String EMAIL_FOOTER_COPYRIGHT = "EMAIL_FOOTER_COPYRIGHT";
	private final static String EMAIL_DISCLAIMER = "EMAIL_DISCLAIMER";
	private final static String EMAIL_SPAM_DISCLAIMER = "EMAIL_SPAM_DISCLAIMER";
	private final static String EMAIL_ADMIN_LABEL = "EMAIL_ADMIN_LABEL";
	private final static String LOGOPATH = "LOGOPATH";
	
	@Inject
	@Qualifier("img")
	private ImageFilePath imageUtils;
	
	/**
	 * Builds generic html email information
	 * @param store
	 * @param messages
	 * @param locale
	 * @return
	 */
	public Map<String, String> createEmailObjectsMap(String contextPath, MerchantStore store, LabelUtils messages, Locale locale){
		
		Map<String, String> templateTokens = new HashMap<String, String>();
		
		String[] adminNameArg = {store.getStorename()};
		String[] adminEmailArg = {store.getStoreEmailAddress()};
		String[] copyArg = {store.getStorename(), DateUtil.getPresentYear()};
		
		templateTokens.put(EMAIL_ADMIN_LABEL, messages.getMessage("email.message.from", adminNameArg, locale));
		templateTokens.put(EMAIL_STORE_NAME, store.getStorename());
		templateTokens.put(EMAIL_FOOTER_COPYRIGHT, messages.getMessage("email.copyright", copyArg, locale));
		templateTokens.put(EMAIL_DISCLAIMER, messages.getMessage("email.disclaimer", adminEmailArg, locale));
		templateTokens.put(EMAIL_SPAM_DISCLAIMER, messages.getMessage("email.spam.disclaimer", locale));
		
		if(store.getStoreLogo()!=null) {
			//TODO revise
			StringBuilder logoPath = new StringBuilder();
			String scheme = Constants.HTTP_SCHEME;
			logoPath.append("<img src='").append(scheme).append("://").append(store.getDomainName()).append(contextPath).append("/").append(imageUtils.buildStoreLogoFilePath(store)).append("' style='max-width:400px;'>");
			templateTokens.put(LOGOPATH, logoPath.toString());
		} else {
			templateTokens.put(LOGOPATH, store.getStorename());
		}

		return templateTokens;
	}

}



```
