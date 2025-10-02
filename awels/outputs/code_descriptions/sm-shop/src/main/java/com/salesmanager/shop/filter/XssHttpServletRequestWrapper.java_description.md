# XssHttpServletRequestWrapper.java

## Review

## 1. Summary  
The `XssHttpServletRequestWrapper` is a thin wrapper around the standard `HttpServletRequest`.  
Its sole purpose is to sanitize all user‑supplied input (headers, parameters, and parameter values) before the application logic consumes them, thereby mitigating Cross‑Site Scripting (XSS) attacks.  

Key components  
* **HttpServletRequestWrapper** – The base class from the Servlet API that allows selective method overriding.  
* **SanitizeUtils.getSafeString()** – A custom utility that presumably escapes dangerous characters or strips malicious payloads.  
* **Overridden methods** – `getHeader`, `getParameter`, and `getParameterValues` – each delegate to the original request and then apply `cleanXSS()`.

Design patterns / frameworks  
* *Decorator pattern* – The wrapper decorates the original request with added sanitization logic.  
* No external frameworks are used; the implementation relies solely on the Servlet API and a user‑defined utility.

---

## 2. Detailed Description  
### Execution flow  
1. **Instantiation** – When a filter or servlet creates `new XssHttpServletRequestWrapper(request)`, the constructor simply calls `super(request)`; no additional state is stored.  
2. **Header access** – The first time `getHeader(name)` is invoked, the wrapper calls the underlying request’s `getHeader`, then passes the result to `cleanXSS()`.  
3. **Parameter access** –  
   * `getParameter(parameter)` retrieves the raw parameter value from the wrapped request, sanitizes it, and returns it.  
   * `getParameterValues(parameter)` handles multi‑valued parameters, sanitizing each entry in the array.  
4. **Other methods** – All other request methods (`getParameterMap()`, `getHeaders()`, etc.) are *not* overridden, so they return the raw data unchanged.  
5. **Cleanup** – No special cleanup is required; the wrapper relies on garbage collection for the underlying request.

### Assumptions & constraints  
* The `SanitizeUtils.getSafeString()` method performs **complete** XSS sanitization for the target environment.  
* The wrapper is applied **once** per request; multiple wrappers could cause redundant or conflicting sanitization.  
* Thread safety is guaranteed by the Servlet container, as the wrapper does not maintain mutable shared state.  

### Architectural choices  
* Keeping the wrapper lightweight (no state, minimal overrides) reduces overhead.  
* The use of a separate sanitization utility keeps the logic decoupled from request handling, enabling reuse elsewhere (e.g., in REST controllers or CLI tools).  
* The decorator pattern ensures backward compatibility: any code that expects an `HttpServletRequest` continues to work.

---

## 3. Functions/Methods  
| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `public XssHttpServletRequestWrapper(HttpServletRequest request)` | Constructor – stores the wrapped request. | `HttpServletRequest request` | A new wrapper instance | None |
| `public String getHeader(String name)` | Retrieves the header value and sanitizes it. | `String name` | Sanitized header value or `null` | None |
| `public String getParameter(String parameter)` | Retrieves a single parameter value and sanitizes it. | `String parameter` | Sanitized value or `null` | None |
| `public String[] getParameterValues(String parameter)` | Retrieves all values for a parameter and sanitizes each. | `String parameter` | Array of sanitized values or `null` | None |
| `private String cleanXSS(String value)` | Delegates to `SanitizeUtils.getSafeString()` to perform the actual escaping/cleaning. | `String value` | Sanitized string | None |

*Reusable utility*: `cleanXSS()` is a thin wrapper around `SanitizeUtils.getSafeString()` and can be reused elsewhere if needed.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard | Part of the Servlet API (Java EE / Jakarta EE). |
| `javax.servlet.http.HttpServletRequestWrapper` | Standard | Also from the Servlet API. |
| `com.salesmanager.shop.utils.SanitizeUtils` | Third‑party (internal) | Custom utility; not part of the JDK or Servlet spec. |
| `com.salesmanager.shop.filter` | Package | The wrapper itself. |

No external libraries (e.g., OWASP ESAPI, Apache Commons Text) are used. All functionality relies on the custom `SanitizeUtils` implementation.

---

## 5. Additional Notes  

### Strengths  
* **Simplicity** – Minimal code with clear intent.  
* **Decoupling** – Sanitization logic lives in a dedicated utility, making it easier to maintain or replace.  
* **Compatibility** – Since it extends `HttpServletRequestWrapper`, it can be plugged into any servlet filter chain.

### Potential Weaknesses / Edge Cases  
1. **Incomplete sanitization** – Overreliance on a single utility method may miss context‑specific filtering (e.g., JSON payloads, XML).  
2. **Missing overrides** – Methods such as `getHeaderNames()`, `getHeaders(String)`, `getParameterMap()`, and `getParameterNames()` are not sanitized. A malicious attacker could bypass filtering by retrieving headers or parameters via these unprotected methods.  
3. **Encoding issues** – If the incoming request uses a non‑standard charset or double‑encoding, `cleanXSS()` might not handle it gracefully.  
4. **Performance overhead** – Sanitizing every header and parameter could impact throughput on high‑traffic endpoints, especially if `getSafeString()` performs regex or DOM parsing.  
5. **Idempotency** – If the wrapper is applied multiple times (e.g., nested filters), repeated sanitization may produce incorrect results (double‑encoding).  
6. **Testing** – No unit tests accompany the wrapper. Without tests, subtle bugs (e.g., null‑pointer exceptions when the underlying request returns `null`) could slip through.

### Recommendations for Improvement  
| Item | Suggested Action |
|------|------------------|
| **Broader method coverage** | Override `getHeaderNames()`, `getHeaders(String)`, `getParameterMap()`, and `getParameterNames()` to ensure all input paths are sanitized. |
| **Use established library** | Consider delegating to OWASP ESAPI, Apache Commons Text (`StringEscapeUtils.escapeHtml4`), or a dedicated XSS filtering library to benefit from community‑maintained, battle‑tested code. |
| **Add tests** | Write unit tests covering all overridden methods, edge cases (null values, empty arrays, multiple values), and potential double‑encoding scenarios. |
| **Performance profiling** | Measure the cost of `cleanXSS()` under load; if necessary, implement caching or a more efficient escaping strategy. |
| **Documentation** | Add Javadoc detailing what `SanitizeUtils.getSafeString()` does, its limitations, and any assumptions about input encoding. |
| **Error handling** | Decide on a strategy for malformed inputs (e.g., malformed Unicode) and document it. |
| **Configuration** | Expose the sanitization strategy via dependency injection so that different environments (dev, test, prod) can use different sanitizers or thresholds. |

Overall, the wrapper serves a clear security purpose but would benefit from a more exhaustive sanitization strategy, better integration with existing libraries, and comprehensive testing to ensure reliability in production.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.filter;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;

import com.salesmanager.shop.utils.SanitizeUtils;

/**
 * Cross Site Scripting filter enforcing html encoding of request parameters
 * @author carlsamson
 *
 */
public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {

	public XssHttpServletRequestWrapper(HttpServletRequest request) {
		super(request);	
		
	}
	

	 
	 @Override
	    public String getHeader(String name) {
	        String value = super.getHeader(name);
	        if (value == null)
	            return null;
	        return cleanXSS(value);
	    }
	 
	 
	    public String[] getParameterValues(String parameter) {
	        String[] values = super.getParameterValues(parameter);
	        if (values == null) {
	            return null;
	        }
	        int count = values.length;
	        String[] encodedValues = new String[count];
	        for (int i = 0; i < count; i++) {
	            encodedValues[i] = cleanXSS(values[i]);
	        }
	        return encodedValues;
	    }
	    
	    @Override
	    public String getParameter(String parameter) {
	        String value = super.getParameter(parameter);
	        if (value == null) {
	            return null;
	        }
	        return cleanXSS(value);
	    }

	    private String cleanXSS(String value) {
	        // You'll need to remove the spaces from the html entities below
	    	return SanitizeUtils.getSafeString(value);
	    }

}



```
