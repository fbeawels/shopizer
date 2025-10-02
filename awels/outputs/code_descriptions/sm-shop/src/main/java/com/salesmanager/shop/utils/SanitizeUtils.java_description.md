# SanitizeUtils.java

## Review

## 1. Summary  
`SanitizeUtils` is a small, utility‑only class that centralises the sanitisation of user‑supplied strings. It serves two purposes:

1. **HTML sanitisation** – Using OWASP Anti‑Samy, it cleans arbitrary HTML according to a custom policy file (`antisamy‑slashdot.xml`).
2. **Request‑parameter filtering** – For query‑string or form‑data values, it removes a hard‑coded blacklist of characters and then XML‑escapes the result.

The class is intentionally non‑instantiable (private constructor) and exposes two static public methods, `getSafeString` and `getSafeRequestParamString`. The design relies on standard Java IO, the Apache Commons Lang utilities, and the OWASP Anti‑Samy library.

---

## 2. Detailed Description  

### 2.1. Core Components  
| Component | Role |
|-----------|------|
| `blackList` | A `List<Character>` that enumerates characters considered unsafe for request parameters. |
| `POLICY_FILE` | Constant holding the Anti‑Samy policy filename. |
| `policy` | Cached instance of `org.owasp.validator.html.Policy`. |
| Static initializer | Loads the policy file from the classpath; fails fast by throwing `ServiceRuntimeException`. |
| `getSafeString` | Accepts raw input, runs Anti‑Samy, and returns cleaned HTML. |
| `getSafeRequestParamString` | Filters a string per the blacklist and then escapes XML 1.1 entities. |

### 2.2. Execution Flow  

1. **Class loading** – The static block executes once. It retrieves `antisamy‑slashdot.xml` via the `Policy` class loader and instantiates a `Policy`. If the file is missing or malformed, a `ServiceRuntimeException` is thrown, preventing any further operation.  
2. **`getSafeString`**  
   * Checks that the policy is not `null` (guard clause).  
   * Creates an `AntiSamy` instance, scans the input, and returns `CleanResults.getCleanHTML()`.  
3. **`getSafeRequestParamString`**  
   * Builds a `StringBuilder` by iterating over the input and appending only characters not present in `blackList`.  
   * Escapes the resulting string using `StringEscapeUtils.escapeXml11`.  

No explicit cleanup is required; resources are managed by the JVM.  

### 2.3. Assumptions & Constraints  

- **Policy Availability** – The policy file must be on the classpath. Any mis‑placement leads to a runtime failure.  
- **Immutability** – The blacklist is static and immutable; any future extension requires code change.  
- **Encoding** – `escapeXml11` assumes the string is in UTF‑8; if a different encoding is used upstream, the output may be incorrect.  
- **Thread‑Safety** – All methods are thread‑safe because they use local variables and immutable static data.  

### 2.4. Architecture & Design Choices  

- **Utility Class Pattern** – The private constructor and all static members enforce statelessness.  
- **Early‑Failure Loading** – The static block ensures the application cannot run with a broken sanitisation policy.  
- **Separation of Concerns** – `getSafeString` focuses on complex HTML sanitisation (Anti‑Samy), while `getSafeRequestParamString` addresses simpler request‑parameter filtering.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getSafeString(String value)` | `public static String getSafeString(String value)` | Cleans arbitrary HTML using Anti‑Samy. | `value`: raw user input | Cleaned HTML (may be empty) | Throws `ServiceRuntimeException` on failure. |
| `getSafeRequestParamString(String value)` | `public static String getSafeRequestParamString(String value)` | Filters characters that are disallowed in request parameters and XML‑escapes the result. | `value`: raw query/form string | Escaped string safe for XML contexts | None. |
| `SanitizeUtils()` (private) | `private SanitizeUtils()` | Prevents instantiation. | None | N/A | None. |

**Reusable / Utility Methods** – The class itself is a collection of reusable utilities; no private helper methods are present beyond the constructor.

---

## 4. Dependencies  

| Library | Purpose | Standard / Third‑Party |
|---------|---------|------------------------|
| `org.owasp.validator.html` (Anti‑Samy) | HTML sanitisation and policy parsing | Third‑Party |
| `org.apache.commons.lang3.StringEscapeUtils` | XML escaping | Third‑Party |
| `org.apache.commons.lang3.StringUtils` | Null/empty checks | Third‑Party |
| `java.io.InputStream` | Reading policy file | Standard |
| `java.util.Arrays`, `java.util.List` | List utilities | Standard |
| `com.salesmanager.shop.store.api.exception.ServiceRuntimeException` | Application‑specific runtime exception | Custom |

The code does not rely on any platform‑specific features; it can run on any JVM with the mentioned libraries in the classpath.

---

## 5. Additional Notes  

### 5.1. Edge Cases & Limitations  

- **Policy File Missing** – The static block will throw an exception at class loading, halting the application. A more graceful fallback or clearer error message could aid debugging.  
- **Blacklist Completeness** – The `blackList` contains duplicate `'%'` and `'\'` entries; duplication has no functional impact but could be cleaned up.  
- **Character Removal vs. Escaping** – For request parameters, the approach removes characters entirely rather than escaping them. This might alter legitimate user input (e.g., URLs containing `?` or `&`). Consider encoding instead of stripping.  
- **XML Escaping vs. HTML Escaping** – `escapeXml11` is used for request parameters; if the target context is HTML, `StringEscapeUtils.escapeHtml4` might be more appropriate.  
- **Unicode Characters** – The blacklist only covers ASCII. Surrogate pairs or non‑ASCII punctuation that could be used maliciously are not filtered.  

### 5.2. Potential Enhancements  

1. **Externalise Blacklist** – Load the blacklist from a properties file or database to allow runtime updates without redeploying.  
2. **Configurable Policy** – Permit specifying a different Anti‑Samy policy per environment or per API endpoint.  
3. **Better Logging** – Capture and log the original and sanitized values for audit purposes.  
4. **Unit Tests** – Add comprehensive tests covering various attack vectors (XSS, CSRF, malformed HTML).  
5. **Refactor** – Extract the character filtering logic into a separate private method or a dedicated class for clarity.  

### 5.3. Security Considerations  

- **OWASP Anti‑Samy** is a mature library for sanitising HTML; using it is a strong practice.  
- The blacklist approach for request parameters is simplistic and may not cover all injection vectors.  
- Ensure the policy file is kept up‑to‑date and validated against the latest OWASP recommendations.  

---

**Verdict:** The class is concise, well‑structured, and leverages established libraries for its core functions. Minor clean‑ups (duplicate blacklist entries, clearer error handling) and a review of the request‑parameter filtering logic would enhance robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import java.io.InputStream;
import java.util.Arrays;
import java.util.List;

import org.apache.commons.lang3.StringEscapeUtils;
import org.apache.commons.lang3.StringUtils;
import org.owasp.validator.html.AntiSamy;
import org.owasp.validator.html.CleanResults;
import org.owasp.validator.html.Policy;

import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;

public class SanitizeUtils {

	/**
	 * should not contain /
	 */
    private static List<Character> blackList = Arrays.asList(';','%', '&', '=', '|', '*', '+', '_',
            '^', '%','$','(', ')', '{', '}', '<', '>', '[',
            ']', '`', '\'', '~','\\', '?','\'');
    
    private final static String POLICY_FILE = "antisamy-slashdot.xml";
    
    private static Policy policy = null;
    
    static { 
		try {
			ClassLoader loader = Policy.class.getClassLoader();
	        InputStream configStream = loader.getResourceAsStream(POLICY_FILE);
			policy = Policy.getInstance(configStream);
	        
		} catch (Exception e) {
			throw new ServiceRuntimeException(e);
		}
    } 

    private SanitizeUtils() {
        //Utility class
    }
    
    public static String getSafeString(String value) {

		try {

			if(policy == null) {
				throw new ServiceRuntimeException("Error in " + SanitizeUtils.class.getName() + " html sanitize utils is null");		}

	        AntiSamy as = new AntiSamy();
	        CleanResults cr = as.scan(value, policy);
	        
	        return cr.getCleanHTML();
	        
		} catch (Exception e) {
			throw new ServiceRuntimeException(e);
		}


    	
    }
    
    
    public static String getSafeRequestParamString(String value) {

    StringBuilder safe = new StringBuilder();
    if(StringUtils.isNotEmpty(value)) {
        // Fastest way for short strings - https://stackoverflow.com/a/11876086/195904
        for(int i=0; i<value.length(); i++) {
            char current = value.charAt(i);
            if(!blackList.contains(current)) {
                safe.append(current);
            }
        }
    }
    return StringEscapeUtils.escapeXml11(safe.toString());
}
    


/*	public static String getSafeString(String value) {
		
		
        //value = value.replaceAll("<", "& lt;").replaceAll(">", "& gt;");
        //value = value.replaceAll("\\(", "& #40;").replaceAll("\\)", "& #41;");
        //value = value.replaceAll("'", "& #39;");
        value = value.replaceAll("eval\\((.*)\\)", "");
        value = value.replaceAll("[\\\"\\\'][\\s]*javascript:(.*)[\\\"\\\']", "\"\"");

        value = value.replaceAll("(?i)<script.*?>.*?<script.*?>", "");
        value = value.replaceAll("(?i)<script.*?>.*?</script.*?>", "");
        value = value.replaceAll("(?i)<.*?javascript:.*?>.*?</.*?>", "");
        value = value.replaceAll("(?i)<.*?\\s+on.*?>.*?</.*?>", "");
        //value = value.replaceAll("<script>", "");
        //value = value.replaceAll("</script>", "");
        
        //return HtmlUtils.htmlEscape(value);	
		
        StringBuilder safe = new StringBuilder();
        if(StringUtils.isNotEmpty(value)) {
            // Fastest way for short strings - https://stackoverflow.com/a/11876086/195904
            for(int i=0; i<value.length(); i++) {
                char current = value.charAt(i);
                if(!blackList.contains(current)) {
                    safe.append(current);
                }
            }
        }
        return StringEscapeUtils.escapeXml11(safe.toString());
	}*/

}



```
