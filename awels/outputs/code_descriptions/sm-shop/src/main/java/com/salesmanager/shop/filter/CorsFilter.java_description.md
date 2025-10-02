# CorsFilter.java

## Review

## 1. Summary  
The `CorsFilter` class is a Spring MVC interceptor that injects **CORS (Cross‑Origin Resource Sharing)** headers into every HTTP response. Its main purpose is to allow public web services to be called from remote hosts by adding the appropriate `Access‑Control‑*` headers.  
Key components:  

| Component | Role |
|-----------|------|
| `CorsFilter` | Extends `HandlerInterceptorAdapter` and overrides `preHandle` to set CORS headers |
| `preHandle` | Executes before a controller method and writes headers to the `HttpServletResponse` |
| Apache Commons Lang (`StringUtils`) | Utility to check if the “origin” header is blank |

The class does **not** implement the `javax.servlet.Filter` interface; it merely uses that import accidentally.

---

## 2. Detailed Description  
### Architecture  
The filter is registered as a Spring MVC interceptor (presumably via `WebMvcConfigurer` or XML). Whenever a request is processed, Spring invokes `preHandle`. The method inspects the request for an `origin` header; if present, that value is used, otherwise `*` (allow all origins) is set.

```java
origin = request.getHeader("origin");
```

Headers written:

| Header | Value |
|--------|-------|
| `Access-Control-Allow-Methods` | `POST, GET, PUT, OPTIONS, DELETE, PATCH` |
| `Access-Control-Allow-Headers` | `X-Auth-Token, Content-Type, Authorization, Cache-Control, X-Requested-With` |
| `Access-Control-Allow-Origin`  | value from request or `*` |

After setting the headers, the method returns `true` so that processing continues.

### Runtime Flow  
1. **Request enters** the Spring MVC dispatcher.  
2. Interceptor chain executes → `CorsFilter.preHandle` runs.  
3. CORS headers are added.  
4. Request proceeds to the handler (controller).  
5. After the controller, the response leaves the container, already bearing the CORS headers.

No special cleanup is required.

### Assumptions & Constraints  
- The application is built on Spring MVC (pre‑WebFlux).  
- It expects all requests to be allowed from any origin (`*`).  
- No validation of the origin value is performed.  
- It ignores pre‑flight (`OPTIONS`) handling nuances such as `Access-Control-Max-Age`.  
- No credentials (`Access-Control-Allow-Credentials`) are set.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `public CorsFilter()` | Default constructor – does nothing. | None | None | None |
| `public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)` | Adds CORS headers to the response before the handler executes. | `request`, `response`, `handler` | `true` (continues processing) | Writes three headers to the HTTP response. |

*Reusable/utility methods*: None.  
*Notes*: The method casts `ServletResponse` to `HttpServletResponse`; this is safe because Spring guarantees the response type.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.web.servlet.handler.HandlerInterceptorAdapter` | Spring MVC (core) | Deprecated in Spring 5.8+ in favor of implementing `HandlerInterceptor` directly or using `WebMvcConfigurer`. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | Provides null‑safe string checks. |
| `javax.servlet.*` | Standard J2EE | Used for request/response interfaces. |

No platform‑specific code; works on any servlet container (Tomcat, Jetty, etc.).

---

## 5. Additional Notes  

### Strengths  
* Simple, clear implementation that satisfies most basic CORS needs.  
* Uses `HandlerInterceptorAdapter` which integrates seamlessly with Spring MVC lifecycle.  
* Avoids custom servlet filter boilerplate.

### Potential Issues / Edge Cases  
1. **`Access-Control-Allow-Origin: *`**  
   * Allows any origin, which may expose sensitive resources.  
   * If the API uses credentials (`Cookie`, `Authorization` header), the browser will ignore `*`; `Access-Control-Allow-Credentials:true` is missing.

2. **Pre‑flight (`OPTIONS`) handling**  
   * The interceptor does not differentiate `OPTIONS` requests.  
   * For true pre‑flight support, one should return immediately with `200 OK` and the correct headers or rely on Spring’s `CorsConfiguration`.

3. **Header Injection on Redirects**  
   * If the controller returns a redirect (`302`), the interceptor still runs, but some frameworks may override headers.

4. **Thread‑Safety & Performance**  
   * No mutable state, so thread‑safe.  
   * Setting headers on every request adds negligible overhead.

5. **Deprecated Base Class**  
   * `HandlerInterceptorAdapter` is deprecated; newer projects should implement `HandlerInterceptor` directly or use the `CorsFilter` provided by Spring.

### Suggested Enhancements  

| Enhancement | Rationale |
|-------------|-----------|
| **Move to Spring’s CORS support** (`WebMvcConfigurer.addCorsMappings`) | Leverages built‑in, configurable CORS handling, including per‑path, per‑method, and `maxAge`. |
| **Validate `Origin` header** | Only allow trusted origins; avoid open `*` in production. |
| **Support credentials** | Add `Access-Control-Allow-Credentials:true` when needed. |
| **Handle pre‑flight** | Return `200 OK` for `OPTIONS` before invoking the handler. |
| **Remove unused imports** | Clean up compiler warnings. |
| **Use `HandlerInterceptor` instead of deprecated adapter** | Future‑proof against Spring upgrades. |

---  

**Bottom line:** The class achieves basic CORS header injection but can be significantly improved by adopting Spring’s built‑in CORS configuration, tightening origin validation, and handling pre‑flight requests properly.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang3.StringUtils;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

public class CorsFilter extends HandlerInterceptorAdapter {

		public CorsFilter() {
			
		}

		/**
		 * Allows public web services to work from remote hosts
		 */
	   public boolean preHandle(
	            HttpServletRequest request,
	            HttpServletResponse response,
	            Object handler) throws Exception {
		   
        	HttpServletResponse httpResponse = (HttpServletResponse) response;
        	
        	String origin = "*";
        	if(!StringUtils.isBlank(request.getHeader("origin"))) {
        		origin = request.getHeader("origin");
        	}
	
	        httpResponse.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE, PATCH");
        	httpResponse.setHeader("Access-Control-Allow-Headers", "X-Auth-Token, Content-Type, Authorization, Cache-Control, X-Requested-With");
        	httpResponse.setHeader("Access-Control-Allow-Origin", origin);
	        
        	return true;
			
		}
}



```
