# CustomerFacade.java

## Review

## 1. Summary
The `CustomerFacade` interface defines a contract for customer‑related operations that are typically invoked by a web shop controller layer. It abstracts business logic around authentication, password recovery, and customer existence checks into a single façade that can be implemented by service classes.  
Key components:  

- **authorize** – validates the current `Principal` against a `Customer`.  
- **requestPasswordReset** – initiates the “forgot password” flow.  
- **verifyPasswordRequestToken** – confirms that a password‑reset token is still valid.  
- **resetPassword** – performs the actual password change.  
- **customerExists** – checks if a user name is already registered in a particular store.

The design follows the **Facade** pattern, offering a simplified interface over potentially complex domain services such as user repositories, token generators, email notifications, etc. The code relies on standard Java (`Principal`) and a handful of domain entities (`Customer`, `MerchantStore`, `Language`), so it is portable across Java EE / Spring containers.

---

## 2. Detailed Description
### Core Components
| Component | Role |
|-----------|------|
| `CustomerFacade` | Public API exposed to controllers and integration layers. |
| Domain entities (`Customer`, `MerchantStore`, `Language`) | Provide contextual data required for the operations. |
| `Principal` | Represents the authenticated user from the servlet security context. |

### Interaction Flow
1. **Authorization**  
   The controller passes the authenticated `Principal` and a `Customer` object to `authorize`.  
   Implementation should compare the principal’s name (or ID) with the customer’s identity, potentially throwing an exception if mismatched.

2. **Password Reset Request**  
   - `requestPasswordReset` is called with the customer’s login, the request’s context path, the merchant store, and language.  
   - The implementation typically:
     - Validates the customer exists.  
     - Generates a unique token (e.g., UUID or JWT).  
     - Persists the token with an expiry timestamp.  
     - Sends an email to the customer containing a reset link that embeds the token.

3. **Token Verification**  
   - `verifyPasswordRequestToken` receives a token string and a store identifier.  
   - Implementation checks the token’s presence, expiry, and association with the correct store.  
   - May return void or throw an exception on failure; the signature currently allows only exceptions for failure.

4. **Password Reset**  
   - `resetPassword` accepts the new password, token, and store.  
   - Implementation validates the token, ensures the password meets security constraints, updates the customer record, and invalidates the token.

5. **Customer Existence Check**  
   - `customerExists` queries the underlying data store (e.g., a DAO or repository) to confirm if the supplied username is already registered for the given store.

### Assumptions & Constraints
- Tokens are one‑time, time‑bound, and store‑scoped.  
- The method signatures do not expose return values for most operations; the implementer is expected to use exceptions for error reporting.  
- The facade does not prescribe persistence or email mechanisms; these are delegated to concrete implementations.  
- `customerContextPath` is used to build the reset link; the facade assumes the caller supplies the correct context.

### Architecture & Design Choices
- **Facade Pattern**: Provides a clean API for the web layer, shielding it from domain complexity.  
- **Stateless Interface**: All methods are pure actions, no state is retained across calls.  
- **Domain‑Driven**: Operates on domain entities (`MerchantStore`, `Language`) rather than generic DTOs.  
- **Extensibility**: New operations (e.g., password policy checks) can be added without affecting existing methods.

---

## 3. Functions/Methods
| Method | Parameters | Return | Purpose & Side‑Effects |
|--------|------------|--------|------------------------|
| `authorize(Customer, Principal)` | `Customer` – user object; `Principal` – authenticated principal | `void` | Verifies that the authenticated principal matches the supplied customer. Should throw an exception if not authorized. |
| `requestPasswordReset(String, String, MerchantStore, Language)` | `customerName` – login; `customerContextPath` – request context; `store` – merchant; `language` – UI language | `void` | Initiates password‑reset flow: generate token, persist, send email. |
| `verifyPasswordRequestToken(String, String)` | `token` – reset token; `store` – merchant identifier | `void` | Confirms the token is valid and belongs to the given store. Throws on invalid/expired. |
| `resetPassword(String, String, String)` | `password` – new password; `token` – reset token; `store` – merchant identifier | `void` | Updates the customer password, invalidates the token, logs the event. |
| `customerExists(String, MerchantStore)` | `userName` – login; `store` – merchant | `boolean` | Checks if a customer exists for the given store. |

**Reusable / Utility Methods**  
The interface itself contains only public operations; any reusable utilities would be implemented in concrete classes or helper services (e.g., token generation, email formatting).  

---

## 4. Dependencies
| Dependency | Category | Remarks |
|------------|----------|---------|
| `java.security.Principal` | Standard Java | Represents authenticated identity. |
| `com.salesmanager.core.model.customer.Customer` | Domain | Custom entity representing a shop customer. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Domain | Encapsulates store‑specific data (ID, name, etc.). |
| `com.salesmanager.core.model.reference.language.Language` | Domain | Used for localisation of emails or UI. |

All dependencies are domain classes from the Sales Manager core module, implying this façade is part of a larger enterprise application. No external third‑party libraries are referenced directly in the interface.

---

## 5. Additional Notes
### Edge Cases & Potential Issues
- **No Return on Success**: All methods except `customerExists` return `void`. If callers need confirmation, the implementation must throw exceptions on error only, which can obscure success states. Consider returning status objects or booleans for clarity.  
- **Token Handling**: The interface assumes token validation logic is internal. Exposing the token lifecycle (generation, expiry) might be helpful for unit tests or integration flows.  
- **Internationalisation**: `Language` is passed only to `requestPasswordReset`; other methods (e.g., email templates) might also need localisation support.  
- **Security**: The `authorize` method should enforce that the principal’s name matches the customer’s login or ID, but the current signature does not guarantee that the caller provides the correct `Customer`. Additional validation in the implementation is critical.  
- **Exception Strategy**: The interface does not specify checked exceptions. Implementations should document what exceptions are thrown (e.g., `AuthenticationException`, `TokenExpiredException`) to aid callers.  

### Future Enhancements
- **Return DTOs**: Methods could return lightweight response objects indicating success, errors, or remaining attempts for token verification.  
- **Asynchronous Email**: Decouple email sending from the `requestPasswordReset` method, perhaps via an event bus.  
- **Rate Limiting**: Add support for limiting password reset requests per IP/user to mitigate abuse.  
- **Password Strength Validation**: Integrate a policy checker that enforces complexity rules before persisting a new password.  
- **Audit Logging**: Expose hooks or events for auditing password reset activities.  

Overall, the interface is concise, clear, and well‑aligned with a typical e‑commerce customer‑management domain. Implementations should focus on robust security, proper exception handling, and clear contract documentation for the consuming layers.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.customer.facade.v1;

import java.security.Principal;

import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;

public interface CustomerFacade {
	
	/** validates username with principal **/
	void authorize(Customer customer,  Principal principal);
	
	/**
	 * 
	 * Forgot password functionality
	 * @param customerName
	 * @param store
	 * @param language
	 */
	void requestPasswordReset(String customerName, String customerContextPath, MerchantStore store, Language language);
	
	/**
	 * Validates if a password request is valid
	 * @param token
	 * @param store
	 */
	void verifyPasswordRequestToken(String token, String store);
	
	
	/**
	 * Reset password
	 * @param password
	 * @param token
	 * @param store
	 */
	void resetPassword(String password, String token, String store);
	
	/**
	 * Check if customer exist
	 * @param userName
	 * @param store
	 * @return
	 */
	boolean customerExists(String userName, MerchantStore store);

}



```
