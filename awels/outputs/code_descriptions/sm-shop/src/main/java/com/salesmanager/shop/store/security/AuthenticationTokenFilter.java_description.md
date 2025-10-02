# AuthenticationTokenFilter.java

## Review

## 1. Summary

`AuthenticationTokenFilter` is a Spring Web `OncePerRequestFilter` that:

- Adds generic CORS headers to every response.
- Extracts the client IP address and stores it in a thread‑local `UserContext`.
- Parses the `Authorization` header for two token formats:
  - `Bearer <jwt>` – used for customer endpoints (`/api/v1/auth`).
  - `FB <token>` – reserved for Facebook login (currently commented out).
- Delegates authentication to one of two `CustomAuthenticationManager` beans depending on the request path:
  - `jwtCustomCustomerAuthenticationManager` for public auth endpoints.
  - `jwtCustomAdminAuthenticationManager` for private admin endpoints.
- Cleans up the `UserContext` after the request completes.

The filter is intended to be declared as a bean (e.g. with `@Component` or in XML) so that Spring injects `@Value` and `@Inject` dependencies.

---

## 2. Detailed Description

### Initialization
- `tokenHeader` is injected from the property `authToken.header`.
- Two `CustomAuthenticationManager` instances are injected (`@Inject`).
- CORS headers are set on every request before any authentication logic.

### Runtime Flow
1. **CORS Header Injection**  
   The filter sets `Access‑Control‑*` headers based on the request’s `Origin` header.

2. **IP Tracking**  
   `GeoLocationUtils.getClientIpAddress(request)` is called to obtain the real client IP (taking proxies into account).  
   A new `UserContext` instance is created and the IP is stored in it.

3. **Request URL Matching**  
   The request URL is inspected for two patterns:
   - `/api/v1/auth` – public authentication endpoints.
   - `/api/v1/private` or `/api/v2/private` – protected admin endpoints.

4. **Token Extraction & Authentication**  
   - The configured header (e.g. `Authorization`) is read.  
   - If it starts with `Bearer `, the customer or admin authentication manager is invoked.  
   - If it starts with `FB `, a (currently commented) Facebook manager would be called.  
   - If no recognizable token is present, a warning is logged but the request proceeds.

5. **Chain Continuation**  
   `chain.doFilter(request, response)` is called, letting the rest of the filter chain and ultimately the target controller handle the request.

6. **Post‑Filter Cleanup**  
   After the chain completes, the `UserContext` thread‑local is closed to free resources.

### Assumptions & Constraints
- The filter assumes that the `tokenHeader` property is set and non‑null.
- The `CustomAuthenticationManager.authenticateRequest` method is responsible for setting authentication in the Spring `SecurityContextHolder` (or rejecting the request via an exception).
- The IP resolution logic (`GeoLocationUtils`) can throw exceptions; these are swallowed with a log entry.
- The filter does not short‑circuit on missing tokens for private endpoints; it simply logs a warning.
- The filter runs for every request; it does not exclude static resources or health‑check endpoints.

### Architectural Notes
- **Separation of Concerns** – Token parsing and authentication are delegated to dedicated managers.  
- **Thread‑Local Context** – `UserContext` is a thread‑local wrapper that holds request‑specific data (IP).  
- **Extensibility** – Adding new token types (e.g. `Bearer ` vs `APIKEY `) would involve a simple if/else branch.  
- **Reusability** – The IP extraction logic could be extracted to a utility method or filter.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain)` | Main filter entry point; orchestrates CORS, IP capture, token parsing, authentication, and cleanup. | `request`, `response`, `chain` | Sets CORS headers, populates `UserContext`, invokes authentication managers, calls `chain.doFilter`, then `postFilter`. |
| `postFilter(HttpServletRequest, HttpServletResponse, FilterChain)` | Performs cleanup after the filter chain. | `request`, `response`, `chain` | Closes `UserContext` thread‑local, logs any errors. |
| `main` – not present. | — | — | — |

Utility helpers are scattered within `doFilterInternal` (e.g. IP extraction, header checks) and could be refactored into separate private methods for clarity.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.web.filter.OncePerRequestFilter` | Spring Framework | Base class for filters. |
| `org.springframework.beans.factory.annotation.Value` | Spring | Injects configuration property. |
| `javax.inject.Inject` | JSR‑330 | Alternative to Spring’s `@Autowired`. |
| `org.apache.commons.lang3.StringUtils` | Apache Commons Lang | Utility for string checks. |
| `org.slf4j.Logger / LoggerFactory` | SLF4J | Logging abstraction. |
| `com.salesmanager.core.model.common.UserContext` | Custom | Thread‑local context holder. |
| `com.salesmanager.shop.store.security.common.CustomAuthenticationManager` | Custom | Performs authentication logic. |
| `com.salesmanager.shop.utils.GeoLocationUtils` | Custom | IP resolution. |

All third‑party libraries are standard (Spring, Commons Lang, SLF4J). Custom classes are part of the `com.salesmanager` code base.

---

## 5. Additional Notes

### Edge Cases & Potential Issues

1. **Missing `tokenHeader`**  
   If `${authToken.header}` is not set, `this.tokenHeader` will be `null`. `request.getHeader(null)` throws a `NullPointerException`. The filter should guard against this.

2. **Preflight OPTIONS Requests**  
   The filter sets CORS headers but still proceeds to authentication logic. Since `Authorization` is usually absent, it logs a warning and continues. This may not be an issue, but some frameworks prefer to short‑circuit and return `200 OK` immediately.

3. **Token Validation Failures**  
   The filter only logs a warning if the header does not match expected prefixes; it does **not** reject the request. If authentication is mandatory for private endpoints, the downstream authentication manager must throw an exception. If it doesn’t, unauthenticated requests will be allowed.

4. **Resource Leak in `UserContext`**  
   `postFilter` attempts to close `UserContext` even if it was never created. While this is harmless, it’s a little inefficient.

5. **Code Duplication**  
   The logic for extracting the token header and checking prefixes is duplicated for public and private endpoints. Refactor into a helper method to reduce duplication.

6. **Hard‑coded URL Checks**  
   Using `String.contains` is fragile (e.g. `/api/v1/authentication` will also match). Consider a regex or Spring `AntPathMatcher`.

7. **Thread‑Safety**  
   `UserContext` is thread‑local; ensure it is correctly cleared even if an exception occurs in authentication.

### Suggested Enhancements

- **Graceful Failure** – If a token is missing or invalid on a protected endpoint, return a `401 Unauthorized` or `403 Forbidden` early instead of continuing the chain.
- **Dynamic Token Manager** – Instead of two hard‑coded managers, maintain a map of endpoint patterns → manager. This scales when adding more auth schemes.
- **Configuration** – Expose the CORS headers via properties for flexibility.
- **Unit Tests** – Add tests covering:
  - Valid bearer token for public/private endpoints.
  - Missing token on protected endpoint.
  - Preflight OPTIONS request.
  - IP extraction with proxies.
- **Logging** – Include request path and client IP in logs for easier debugging.
- **Exception Handling** – Wrap `authenticateRequest` in a try/catch that returns a meaningful HTTP error rather than rethrowing as a generic `ServletException`.

Overall, the filter implements the core responsibilities cleanly but would benefit from defensive coding, reduced duplication, and more explicit error handling to strengthen security and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.security;

import java.io.IOException;
import java.util.Enumeration;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import javax.inject.Inject;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.filter.OncePerRequestFilter;

import com.salesmanager.core.model.common.UserContext;
import com.salesmanager.shop.store.security.common.CustomAuthenticationManager;
import com.salesmanager.shop.utils.GeoLocationUtils;


public class AuthenticationTokenFilter extends OncePerRequestFilter {


	private static final Logger LOGGER = LoggerFactory.getLogger(AuthenticationTokenFilter.class);

    
    @Value("${authToken.header}")
    private String tokenHeader;
    
    private final static String BEARER_TOKEN ="Bearer ";
    
    private final static String FACEBOOK_TOKEN ="FB ";
    
    //private final static String privateApiPatternString = "/api/v*/private";
    
    //private final static Pattern pattern = Pattern.compile(privateApiPatternString);
    

    
    @Inject
    private CustomAuthenticationManager jwtCustomCustomerAuthenticationManager;
    
    @Inject
    private CustomAuthenticationManager jwtCustomAdminAuthenticationManager;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        

    	String origin = "*";
    	if(!StringUtils.isBlank(request.getHeader("origin"))) {
    		origin = request.getHeader("origin");
    	}
    	//in flight
    	response.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE, PATCH");
    	response.setHeader("Access-Control-Allow-Origin", origin);
    	response.setHeader("Access-Control-Allow-Headers", "X-Auth-Token, Content-Type, Authorization, Cache-Control, X-Requested-With");
    	response.setHeader("Access-Control-Allow-Credentials", "true");

    	try {
    		
    		String ipAddress = GeoLocationUtils.getClientIpAddress(request);
    		
    		UserContext userContext = UserContext.create();
    		userContext.setIpAddress(ipAddress);
    		
    	} catch(Exception s) {
    		LOGGER.error("Error while getting ip address ", s);
    	}
    	
    	String requestUrl = request.getRequestURL().toString();


    	if(requestUrl.contains("/api/v1/auth")) {
    		//setHeader(request,response);   	
	    	final String requestHeader = request.getHeader(this.tokenHeader);//token
	    	
	    	try {
		        if (requestHeader != null && requestHeader.startsWith(BEARER_TOKEN)) {//Bearer
		        	
		        	jwtCustomCustomerAuthenticationManager.authenticateRequest(request, response);
	
		        } else if(requestHeader != null && requestHeader.startsWith(FACEBOOK_TOKEN)) {
		        	//Facebook
		        	//facebookCustomerAuthenticationManager.authenticateRequest(request, response);
		        } else {
		        	LOGGER.warn("couldn't find any authorization token, will ignore the header");
		        }
	        
	    	} catch(Exception e) {
	    		throw new ServletException(e);
	    	}
    	}
 
    	
    	if(requestUrl.contains("/api/v1/private") || requestUrl.contains("/api/v2/private")) {
    		
    		//setHeader(request,response);  
    		
    		Enumeration<String> headers = request.getHeaderNames();
    		//while(headers.hasMoreElements()) {
    			//LOGGER.debug(headers.nextElement());
    		//}

	    	final String requestHeader = request.getHeader(this.tokenHeader);//token
	    	
	    	try {
		        if (requestHeader != null && requestHeader.startsWith(BEARER_TOKEN)) {//Bearer
		        	
		        	jwtCustomAdminAuthenticationManager.authenticateRequest(request, response);
	
		        } else {
		        	LOGGER.warn("couldn't find any authorization token, will ignore the header, might be a preflight check");
		        }
	        
	    	} catch(Exception e) {
	    		throw new ServletException(e);
	    	}
    	}

        chain.doFilter(request, response);
        postFilter(request, response, chain);
    }
    
    
    private void postFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
    	
    	try {
    		
    		UserContext userContext = UserContext.getCurrentInstance();
    		if(userContext!=null) {
    			userContext.close();
    		}
    		
    	} catch(Exception s) {
    		LOGGER.error("Error while getting ip address ", s);
    	}
    	
    }


}



```
