# ServiceRequestCriteriaBuilderUtils.java

## Review

## 1. Summary

`ServiceRequestCriteriaBuilderUtils` is a Spring‑based helper that populates domain‑specific *Criteria* objects from HTTP request parameters.  
It exposes two public entry points:

| Method | Purpose |
|--------|---------|
| `buildRequestCriterias` | Binds request parameters to any `Criteria` implementation using a field‑name mapping. |
| `buildRequest` (deprecated) | Builds a `MerchantStoreCriteria` from a DataTables‑style request. |

The class relies on Spring’s `PropertyAccessorFactory` to set fields directly, bypassing setters, and uses SLF4J for logging (though `System.out` is still present).

---

## 2. Detailed Description

### Core Workflow

1. **Validation** – `buildRequestCriterias` first ensures the supplied `criteria` instance is non‑null, throwing a `RestApiException` if it isn’t.
2. **Parameter Mapping** – Iterates over `mappingFields` (a `Map<String,String>` mapping request param names to field names) and for each entry:
   - Calls `setValue` to pull the parameter value from the `HttpServletRequest` and write it into the `criteria` instance.
3. **Return** – The fully populated `criteria` instance is returned.

`setValue` performs the heavy lifting:

| Step | Action |
|------|--------|
| 1 | Creates a `PropertyAccessor` that operates directly on the target object’s fields. |
| 2 | Retrieves the raw string value for the requested parameter. |
| 3 | If present, uses the accessor to write the value directly into the field identified by the mapping. |
| 4 | Any exception is wrapped and re‑thrown as a generic `Exception`. |

The deprecated `buildRequest` method demonstrates a hand‑coded approach for `MerchantStoreCriteria`, handling sorting, pagination, boolean flags, and a search string.

### Design Choices & Assumptions

- **Direct Field Access** – Bypasses JavaBean mutators to avoid complex logic in setters, but this ties the implementation to field names rather than exposed properties.
- **Type Handling** – All values are taken as strings; no conversion is performed beyond what `PropertyAccessor` may implicitly do (usually just assignment).
- **Error Handling** – Exceptions in `setValue` are swallowed in the stream loop (`e.printStackTrace()`), potentially leaving the caller unaware of binding failures.
- **Logging** – Uses SLF4J but falls back to `System.out.println` for debugging.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|------------|---------|--------|---------|--------------|
| `buildRequestCriterias` | `public static Criteria buildRequestCriterias(Criteria, Map<String,String>, HttpServletRequest) throws RestApiException` | Populates any `Criteria` implementation from request params. | `criteria` (must be instantiated), `mappingFields` (param → field), `request` | Same `criteria` instance with fields set | Throws `RestApiException` if `criteria` is null. |
| `setValue` | `private static void setValue(Criteria, HttpServletRequest, String, String) throws Exception` | Helper that sets a single field on the criteria. | `criteria`, `request`, `parameterName`, `setterValue` (field name) | None | Writes directly to the field; logs attempt; throws generic `Exception` on failure. |
| `buildRequest` *(deprecated)* | `public static Criteria buildRequest(Map<String,String>, HttpServletRequest)` | Creates a `MerchantStoreCriteria` from DataTables parameters. | `mappingFields`, `request` | `MerchantStoreCriteria` instance | Sets fields such as `orderBy`, `criteriaOrderByField`, `name`, `retailers`, `stores`, `search`. |

### Utility / Reusable Parts

- `PropertyAccessorFactory.forDirectFieldAccess` is the key mechanism; it could be exposed as a small helper for other builders.
- The mapping approach (request key → field name) is generic and can be reused for other DTOs.

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Java EE / Jakarta | Standard servlet API. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | Provides `isBlank` checks. |
| `org.slf4j.Logger` / `LoggerFactory` | Third‑party | Standard logging abstraction. |
| `org.springframework.beans.PropertyAccessor` / `PropertyAccessorFactory` | Spring | Enables direct field manipulation. |
| `com.salesmanager.core.model.common.Criteria` & related | In‑house | Domain model for query criteria. |
| `com.salesmanager.shop.store.api.exception.RestApiException` | In‑house | Custom API exception. |

No heavy external frameworks are used beyond Spring Core and SLF4J.

---

## 5. Additional Notes

### Edge Cases & Current Limitations

| Scenario | Current Behavior | Suggested Fix |
|----------|------------------|---------------|
| `mappingFields` contains a key that is **not** present in the request | The method simply returns `null` for that key (no error) | Consider logging a warning if a mapping exists but the request param is missing. |
| Request param value is an empty string (`""`) | `PropertyAccessor` will set the field to `""`. | Decide whether empty strings should be treated as `null`. |
| Boolean parsing (`Boolean.valueOf`) | Accepts only `"true"`/`"false"` case‑insensitively. | Add validation and default values; log or throw for unexpected values. |
| `setValue` catches `Exception` and prints stack trace | Swallows error, leaving the calling method unaware | Re‑throw a custom exception (e.g., `RestApiException`) to surface the issue. |
| `System.out.println` inside `setValue` | Not thread‑safe and pollutes logs | Replace with `LOGGER.debug`. |
| No type conversion (int, date, etc.) | All values are treated as strings | Add optional converters or use Spring’s `ConversionService`. |
| No support for nested objects | Only flat fields can be set | Extend the mapping to support nested paths (e.g., `address.street`). |

### Future Enhancements

1. **Strong Typing** – Introduce a `Map<String, BiConsumer<Criteria, String>>` to allow custom converters per field.
2. **Validation** – Use JSR‑380 (`javax.validation`) annotations on Criteria classes and trigger validation after binding.
3. **Error Propagation** – Wrap all binding failures in a single `RestApiException` with detailed context.
4. **Logging** – Replace all `System.out` calls with proper SLF4J levels (`debug`, `info`, `error`).
5. **Extensibility** – Provide a fluent builder API for criteria objects, reducing boilerplate in the deprecated method.
6. **Unit Tests** – Add comprehensive tests covering various mapping scenarios, null handling, and exception paths.

Overall, the utility is concise and serves its purpose, but the current error handling and logging strategies limit its robustness in a production environment. Refactoring towards a more declarative, type‑safe, and testable design would greatly improve maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.PropertyAccessor;
import org.springframework.beans.PropertyAccessorFactory;

import com.salesmanager.core.model.common.Criteria;
import com.salesmanager.core.model.common.CriteriaOrderBy;
import com.salesmanager.core.model.merchant.MerchantStoreCriteria;
import com.salesmanager.shop.store.api.exception.RestApiException;

public class ServiceRequestCriteriaBuilderUtils {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ServiceRequestCriteriaBuilderUtils.class);
	
	/**
	 * Binds request parameter values to specific request criterias
	 * @param criteria
	 * @param mappingFields
	 * @param request
	 * @return
	 * @throws Exception
	 */
	public static Criteria buildRequestCriterias(Criteria criteria, Map<String, String> mappingFields, HttpServletRequest request) throws RestApiException {
		
			if(criteria == null)
				throw new RestApiException("A criteria class type must be instantiated");
	
			mappingFields.keySet().stream().forEach(p -> {
				try {
					setValue(criteria, request, p, mappingFields.get(p));
				} catch (Exception e) {
					e.printStackTrace();
				}
			});
			return criteria;
		

		
	}
	
	private static void setValue(Criteria criteria, HttpServletRequest request, String parameterName, String setterValue) throws Exception {
		
		
		try {
			
			PropertyAccessor criteriaAccessor = PropertyAccessorFactory.forDirectFieldAccess(criteria);
			
			
			String parameterValue = request.getParameter(parameterName);
			if(parameterValue == null) return;
			// set the property directly, bypassing the mutator (if any)
			//String setterName = "set" + WordUtils.capitalize(setterValue);
			String setterName = setterValue;
			System.out.println("Trying to do this binding " + setterName + "('" + parameterValue + "') on " + criteria.getClass());
			criteriaAccessor.setPropertyValue(setterName, parameterValue);
		
		} catch(Exception e) {
			throw new Exception("An error occured while parameter bindding", e);
		}
		
		
	}
		   
  /** deprecated **/
  public static Criteria buildRequest(Map<String, String> mappingFields, HttpServletRequest request) {
    
    /**
     * Works assuming datatable sends query data
     */
    MerchantStoreCriteria criteria = new MerchantStoreCriteria();

    String searchParam = request.getParameter("search[value]");
    String orderColums = request.getParameter("order[0][column]");

    if (!StringUtils.isBlank(orderColums)) {
      String columnName = request.getParameter("columns[" + orderColums + "][data]");
      String overwriteField = columnName;
      if (mappingFields != null && mappingFields.get(columnName) != null) {
        overwriteField = mappingFields.get(columnName);
      }
      criteria.setCriteriaOrderByField(overwriteField);
      criteria.setOrderBy(
          CriteriaOrderBy.valueOf(request.getParameter("order[0][dir]").toUpperCase()));
    }
    
    String storeName = request.getParameter("storeName");
    criteria.setName(storeName);
    
    String retailers = request.getParameter("retailers");
    String stores = request.getParameter("stores");
    
    try {
    	boolean retail = Boolean.valueOf(retailers);
    	boolean sto = Boolean.valueOf(stores);

        criteria.setRetailers(retail);
        criteria.setStores(sto);
    } catch(Exception e) {
    	LOGGER.error("Error parsing boolean values",e);
    }
    
    criteria.setSearch(searchParam);

    return criteria;
    
  }

}



```
