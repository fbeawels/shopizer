# MerchantStoreArgumentResolver.java

## Review

## 1. Summary  
**Purpose & Core Functionality**  
`MerchantStoreArgumentResolver` is a Spring MVC component that automatically injects a `MerchantStore` instance into controller methods. When a controller declares a parameter of type `MerchantStore`, this resolver fetches the appropriate store based on the incoming request and authorizes the user against it.

**Key Components**  
- **`supportsParameter`** – Declares that the resolver applies only to `MerchantStore` parameters.  
- **`resolveArgument`** – Core logic:  
  1. Determine the store code from the request parameter `store` (fallback to `DEFAULT_STORE`).  
  2. Retrieve the `MerchantStore` from `StoreFacade`.  
  3. Verify the request is authorized by `UserFacade`.  
  4. Return the store or throw `UnauthorizedException`.

**Design Patterns & Frameworks**  
- Implements Spring’s `HandlerMethodArgumentResolver` interface – a strategy pattern that allows pluggable argument resolution.  
- Uses dependency injection (`@Autowired`) to obtain `StoreFacade` and `UserFacade`.  
- Employs SLF4J for logging and Apache Commons’ `StringUtils` for null‑safe string checks.

---

## 2. Detailed Description  
### Flow of Execution  
1. **Invocation** – Spring MVC intercepts a request that maps to a controller method containing a `MerchantStore` parameter.  
2. **supportsParameter** – The resolver verifies that the parameter type matches `MerchantStore`.  
3. **resolveArgument** –  
   - Extracts the `store` request parameter; if absent or blank, uses the constant `DEFAULT_STORE`.  
   - Calls `storeFacade.get(storeValue)` to fetch the store model.  
   - Retrieves the raw `HttpServletRequest` to access the request URI.  
   - Calls `userFacade.authorizeStore` to ensure the current user may access the identified store.  
   - Logs the authorization result at debug level.  
   - If unauthorized, throws `UnauthorizedException`; otherwise returns the `MerchantStore` for method injection.  

### Assumptions & Constraints  
- The request always contains an HTTP servlet (i.e., the application is deployed in a servlet container).  
- `storeFacade.get` never returns `null` for a known store code; otherwise a `NullPointerException` would occur.  
- The `UserFacade.authorizeStore` method handles all business‑level security checks.  
- Caching of store data is acknowledged as a TODO; currently each request triggers a DB call.  

### Architecture & Design Choices  
- **Separation of Concerns** – Business logic (store lookup, authorization) is delegated to dedicated facades.  
- **Extensibility** – By implementing `HandlerMethodArgumentResolver`, additional arguments can be injected similarly.  
- **Thread Safety** – All fields are immutable after construction; the resolver itself is stateless and safe for concurrent use.  

---

## 3. Functions/Methods  
| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `boolean supportsParameter(MethodParameter)` | Indicates applicability to a method parameter. | `parameter` – candidate method parameter | `true` if parameter type is `MerchantStore`; else `false` | None |
| `Object resolveArgument(MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory)` | Resolves the argument for controller methods. | * `parameter` – method parameter info (unused except type check) <br> * `mavContainer` – MVC container (unused) <br> * `webRequest` – current request context <br> * `binderFactory` – data binder factory (unused) | `MerchantStore` instance or throws `UnauthorizedException` | Logs debug info; may throw exception |

**Utility** – The resolver itself contains no reusable helper methods, but it relies on external facades that encapsulate complex logic.

---

## 4. Dependencies  
| Library / Framework | Nature | Usage |
|---------------------|--------|-------|
| **Spring MVC** (`org.springframework.web.method.support.HandlerMethodArgumentResolver`, `@Component`, `@Autowired`) | Third‑party (Spring) | Core to MVC argument resolution and dependency injection. |
| **SLF4J** (`org.slf4j.Logger`, `LoggerFactory`) | Third‑party | Logging. |
| **Apache Commons Lang** (`org.apache.commons.lang3.StringUtils`) | Third‑party | Safe string checks (`isNotBlank`). |
| **Java EE / Servlet API** (`javax.servlet.http.HttpServletRequest`) | Standard | Access to request URI. |
| **SalesManager Core / Store Facades** (`StoreFacade`, `UserFacade`, `MerchantStore`) | Project‑specific | Business logic for store retrieval and authorization. |
| **Custom Exception** (`UnauthorizedException`) | Project‑specific | Signals unauthorized access. |

All dependencies are either part of the standard Java EE environment or well‑known open‑source libraries. No platform‑specific assumptions beyond a servlet container.

---

## 5. Additional Notes  
### Strengths  
- **Clear Separation** – Delegates heavy lifting to facades, keeping the resolver lightweight.  
- **Extensibility** – The resolver can be easily swapped or extended without modifying controllers.  
- **Logging** – Provides diagnostic information useful during debugging.  

### Weaknesses & Edge Cases  
1. **Null `MerchantStore` Handling** – If `storeFacade.get` returns `null` (e.g., an invalid store code), the resolver will throw a `NullPointerException` when accessing `storeModel.getCode()`. A defensive check and a more descriptive exception would improve robustness.  
2. **Caching** – Each request triggers a lookup; the TODO comment indicates a missing cache layer. Without caching, performance may suffer under load.  
3. **Authorization Granularity** – The resolver calls `authorizeStore` with only the request URI; it does not consider HTTP method, query parameters, or request body. Future enhancements could expose a richer security context.  
4. **Parameter Naming** – The constant `REQUEST_PARAMATER_STORE` has a typo (`PARAMATER`). Renaming to `REQUEST_PARAMETER_STORE` would improve readability.  
5. **Constructor Injection** – Using field injection (`@Autowired`) hampers testability. Switching to constructor injection is recommended.  

### Potential Enhancements  
- **Implement Caching** – Cache `MerchantStore` instances in a `ConcurrentHashMap` or integrate with Spring Cache abstraction.  
- **Return a Domain‑Specific Exception** – Instead of a generic `UnauthorizedException`, create a `StoreAuthorizationException` that includes the store code.  
- **Refactor to Filter** – As noted in the comment, moving the authorization logic to a servlet filter could reduce repeated calls across multiple endpoints.  
- **Unit Tests** – Add tests covering all branches: missing store parameter, invalid store, authorized vs. unauthorized scenarios.  
- **Logging Levels** – Consider moving the debug log to `trace` for high‑volume environments or expose a configurable property to toggle it.  

Overall, the resolver is well‑structured, adheres to Spring conventions, and clearly delineates responsibilities. Addressing the outlined edge cases and refactoring suggestions would elevate its resilience and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.application.config;

import static com.salesmanager.core.business.constants.Constants.DEFAULT_STORE;

import java.util.Optional;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.shop.store.api.exception.UnauthorizedException;
import com.salesmanager.shop.store.controller.store.facade.StoreFacade;
import com.salesmanager.shop.store.controller.user.facade.UserFacade;

@Component
public class MerchantStoreArgumentResolver implements HandlerMethodArgumentResolver {

	private static final Logger LOGGER = LoggerFactory.getLogger(MerchantStoreArgumentResolver.class);
	public static final String REQUEST_PARAMATER_STORE = "store";

	@Autowired
	private StoreFacade storeFacade;

	@Autowired
	private UserFacade userFacade;

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.getParameterType().equals(MerchantStore.class);
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
		String storeValue = Optional.ofNullable(webRequest.getParameter(REQUEST_PARAMATER_STORE))
				.filter(StringUtils::isNotBlank).orElse(DEFAULT_STORE);
		// todo get from cache
		MerchantStore storeModel = storeFacade.get(storeValue);

		HttpServletRequest httpServletRequest = webRequest.getNativeRequest(HttpServletRequest.class);

		// TODO Move to an api filter
		// authorize request
		boolean authorized = userFacade.authorizeStore(storeModel, httpServletRequest.getRequestURI());
		LOGGER.debug("is request authorized {} for {} and store {}", authorized, httpServletRequest.getRequestURI(),
				storeModel.getCode());
		if(!authorized){
			throw new UnauthorizedException("Cannot authorize user for store " + storeModel.getCode());
		}
		return storeModel;
	}
}



```
