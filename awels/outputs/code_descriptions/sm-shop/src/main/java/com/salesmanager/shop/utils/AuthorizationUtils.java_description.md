# AuthorizationUtils.java

## Review

## 1. Summary
`AuthorizationUtils` is a Spring‑managed component that centralises common authorization logic for REST APIs in the *salesmanager* shop module.  
- **Purpose** – Validate that a request is authenticated and that the authenticated user is authorised to perform the requested action on a given `MerchantStore`.  
- **Key components**  
  - `UserFacade` – a façade that exposes user‑management and group‑membership operations.  
  - `Constants.GROUP_SUPERADMIN` – a hardcoded role that bypasses store‑level checks.  
- **Design patterns** – It follows the **Facade** pattern (wrapping `UserFacade`), the **Strategy** pattern is implied by injecting the façade (can be swapped), and the **Utility** pattern for static‑like helper methods.  
- **Frameworks/libraries** – Spring (`@Component`, dependency injection) and Java SE (`List`, `Arrays`). No external auth libraries are used directly.

---

## 2. Detailed Description
### Flow of execution
1. **Authentication**  
   `authenticatedUser()` asks `UserFacade` for the currently authenticated username.  
   If the result is `null`, an `UnauthorizedException` (a runtime exception) is thrown, signalling an unauthenticated request.

2. **Authorization**  
   `authorizeUser(...)` performs two checks:  
   - **Group check** – the user must belong to at least one of the supplied `roles`.  
   - **Store check** – if the user is *not* a super‑admin, they must be authorised to access the requested store (`store.getCode()`).  
   Failure of either check results in an `UnauthorizedException` with an informative message.

### Dependencies and assumptions
- `UserFacade` is expected to throw its own exceptions or return booleans.  
- The method assumes that the `roles` list contains **exact** role identifiers (e.g. `"ADMIN"`).  
- `Constants.GROUP_SUPERADMIN` is a static constant; adding more privileged groups would require code changes.  
- No transactional context is involved; the component purely performs look‑ups.

### Architecture & design choices
- **Stateless component** – no fields beyond the injected facade; safe for concurrent use.  
- **Separation of concerns** – authentication logic is isolated from role/store checks, making unit‑testing easier.  
- **Hardcoded role** – a design compromise: super‑admin bypass is implemented inline instead of a dedicated policy engine.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `authenticatedUser()` | Retrieve the username of the current authenticated user. | none | `String` | Throws `UnauthorizedException` if no user is found. |
| `authorizeUser(String authenticatedUser, List<String> roles, MerchantStore store)` | Validate that the user can act on the provided store with the given roles. | `authenticatedUser` – username<br>`roles` – list of required roles<br>`store` – target `MerchantStore` | none | Throws `UnauthorizedException` on failure. |

**Reusable utility methods**  
- The logic itself is encapsulated; the class could be used by any controller or service that needs to validate a request before proceeding.

---

## 4. Dependencies
| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `Spring Framework` (`@Component`, `@Inject`) | Third‑party | Provides DI and component lifecycle. |
| `Java SE` (`java.util.List`, `java.util.Arrays`) | Standard | Basic collections utilities. |
| `salesmanager.core.model.merchant.MerchantStore` | Application | Domain entity representing a store. |
| `salesmanager.shop.constants.Constants` | Application | Holds static role constants. |
| `salesmanager.shop.store.api.exception.UnauthorizedException` | Application | Runtime exception indicating lack of auth. |
| `salesmanager.shop.store.controller.user.facade.UserFacade` | Application | Facade for user/role queries. |

No platform‑specific (e.g., JPA, security frameworks) or external API dependencies are required.

---

## 5. Additional Notes
### Strengths
- **Clarity** – Each method does a single, well‑named task.  
- **Testability** – The `UserFacade` can be mocked; unit tests can cover authentication and various authorization scenarios.  
- **Thread safety** – Stateless and immutable once constructed.

### Weaknesses / Edge Cases
1. **Role list handling** – `authorizedGroup` is called once, but the method doesn’t check the result; it simply delegates. If that method throws, the exception will bubble up, but a null or empty list may silently pass.  
2. **Super‑admin bypass** – Hardcoded in the method. Adding new privileged roles would require code changes.  
3. **Exception messages** – For the store check, a detailed message is provided; for the role check, no message is thrown (presumably handled by `authorizedGroup`). Consistency would improve debugging.  
4. **Internationalisation** – Error messages are hard‑coded in English; could be extracted to message bundles.  
5. **Extensibility** – If the system evolves to support multi‑tenant groups or hierarchical permissions, the current implementation will struggle.  

### Suggested Enhancements
- **Return detailed status** – Instead of throwing directly inside, return a status object or use a dedicated `AuthorizationService` that can be chained.  
- **Configuration‑driven super roles** – Load the super‑admin role(s) from configuration or database.  
- **Centralised exception handling** – Create a single `AuthorizationException` and let a global exception handler translate it into a 401/403 response.  
- **Logging** – Add logging for failed authorizations to aid audit trails.  
- **Unit tests** – Ensure coverage for all branches (authenticated, unauthenticated, role missing, store missing, super‑admin).  

Overall, the component is concise and functional for its current scope, but future growth may necessitate a more flexible authorization framework.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import java.util.Arrays;
import java.util.List;

import javax.inject.Inject;

import org.springframework.stereotype.Component;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.shop.constants.Constants;
import com.salesmanager.shop.store.api.exception.UnauthorizedException;
import com.salesmanager.shop.store.controller.user.facade.UserFacade;

/**
 * Performs authorization check for REST Api
 * - check if user is in role
 * - check if user can perform actions on marchant
 * @author carlsamson
 *
 */
@Component
public class AuthorizationUtils {
	
	@Inject
	private UserFacade userFacade;
	
	public String authenticatedUser() {
		String authenticatedUser = userFacade.authenticatedUser();
		if (authenticatedUser == null) {
			throw new UnauthorizedException();
		}
		return authenticatedUser;
	}
	
	public void authorizeUser(String authenticatedUser, List<String> roles, MerchantStore store) {
		userFacade.authorizedGroup(authenticatedUser, roles);
		if (!userFacade.userInRoles(authenticatedUser, Arrays.asList(Constants.GROUP_SUPERADMIN))) {
			if (!userFacade.authorizedStore(authenticatedUser, store.getCode())) {
				throw new UnauthorizedException("Operation unauthorized for user [" + authenticatedUser
						+ "] and store [" + store.getCode() + "]");
			}
		}
	}

}



```
