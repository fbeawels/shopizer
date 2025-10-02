# CustomerPassword.java

## Review

## 1. Summary  
**Purpose**  
`CustomerPassword` is a simple *Plain Old Java Object* (POJO) that represents the data required when a customer changes their password. It is used as a data transfer object (DTO) in a web‑application context (likely Spring MVC or Spring Boot) and is subject to Java Bean Validation (JSR‑303).

**Key Components**  
| Component | Role |
|-----------|------|
| `currentPassword` | Holds the user’s existing password (used for authentication before allowing a change). |
| `password` | New password entered by the user. |
| `checkPassword` | Confirmation of the new password. |
| `@FieldMatch` | Custom cross‑field validator that ensures `password` and `checkPassword` are identical. |
| `@NotEmpty` / `@Size` | Built‑in Bean Validation constraints that enforce non‑emptiness and minimum length. |

**Design Patterns / Libraries**  
* **DTO / Value Object pattern** – the class contains only state, no business logic.  
* **Java Bean Validation (JSR‑303)** – annotations such as `@NotEmpty` and `@Size` are part of the standard.  
* **Custom Annotation (`@FieldMatch`)** – indicates that the project has its own validator implementation to compare two fields.  
* **Serializable** – makes the object safe to transmit or cache, a common requirement for DTOs in distributed systems.

---

## 2. Detailed Description  
The class is used in the customer‑password‑change flow:

1. **Reception** – A controller method receives a POST request with the password change form.  
2. **Binding** – The request payload is mapped to an instance of `CustomerPassword` via a data‑binding framework (e.g., Spring’s `@ModelAttribute` or `@RequestBody`).  
3. **Validation** – The JSR‑303 validator processes the annotations:
   * `currentPassword` must not be empty (`@NotEmpty`).  
   * `password` and `checkPassword` must each be at least 6 characters (`@Size(min=6)`).
   * `@FieldMatch` validates that the two new‑password fields are identical.
4. **Business Logic** – If validation passes, the controller delegates to a service that checks whether `currentPassword` matches the stored password, hashes `password`, and persists the change.  
5. **Cleanup** – After the request is processed, the instance is garbage‑collected; no explicit cleanup is required.

**Assumptions & Constraints**  
* The password policy only requires a minimum length of 6 characters; no complexity rules are enforced here.  
* The `currentPassword` field is not validated for length or format – it’s presumed to be checked against the stored hash in service logic.  
* The custom `@FieldMatch` annotation must be correctly implemented and registered with the validation framework; otherwise, the cross‑field check will be ineffective.  
* The class assumes a standard `message` source (e.g., `messages.properties`) where the keys `currentpassword.not.empty`, `newpassword.not.empty`, `repeatpassword.not.empty`, and `password.notequal` are defined.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `getCurrentPassword()` | Getter for `currentPassword`. | None | `String` | None |
| `setCurrentPassword(String)` | Setter for `currentPassword`. | `currentPassword` | `void` | Updates internal state |
| `getPassword()` | Getter for `password`. | None | `String` | None |
| `setPassword(String)` | Setter for `password`. | `password` | `void` | Updates internal state |
| `getCheckPassword()` | Getter for `checkPassword`. | None | `String` | None |
| `setCheckPassword(String)` | Setter for `checkPassword`. | `checkPassword` | `void` | Updates internal state |

*No additional utility or reusable methods are present, which is appropriate for a simple DTO.*

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.io.Serializable` | JDK standard | Enables object serialization (e.g., HTTP session storage). |
| `javax.validation.constraints.*` | JSR‑303 (Bean Validation) | Standard annotations (`@NotEmpty`, `@Size`). |
| `com.salesmanager.shop.validation.FieldMatch` | Custom | Must be implemented elsewhere in the codebase; provides cross‑field validation. |

The class itself is framework‑agnostic but is intended for use in a Spring‑based web application (given the typical patterns of DTO + validation).

---

## 5. Additional Notes  

### Strengths  
* **Clear separation of concerns** – the class contains only data and validation rules.  
* **Reusability** – can be reused across multiple endpoints (e.g., password reset, profile update).  
* **Localization support** – messages are referenced via keys, allowing easy internationalization.

### Potential Edge Cases / Improvements  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Missing length/complexity checks on `currentPassword`** | Users could submit very short current passwords that still pass `@NotEmpty`. | Add `@Size(min=6)` or a custom password strength validator if needed. |
| **No `@NotEmpty` on `password` and `checkPassword`** | The `@Size(min=6)` constraint still allows `null` values, which may bypass the length check in some implementations. | Add `@NotEmpty` to both fields to guard against `null`. |
| **`FieldMatch` implementation** | If the custom validator is not correctly wired, the equality check may fail silently. | Verify that `FieldMatch` registers its validator via `ConstraintValidator` and that it is listed in `META-INF/services/...`. |
| **Potential NPE in custom validator** | If either field is `null`, the validator may throw an exception. | Ensure the validator handles `null` gracefully (e.g., treat `null` as a mismatch). |
| **No `toString()`, `equals()`, or `hashCode()`** | Debugging or logging the object prints the class name only. | Override `toString()` (excluding sensitive data) for better logging. |

### Future Enhancements  

1. **Add password strength validation** – e.g., enforce a mix of letters, numbers, and special characters.  
2. **Integrate with a centralized `PasswordPolicy` service** – make validation rules configurable.  
3. **Unit‑test the custom `@FieldMatch` annotation** – ensure it behaves as expected with various inputs.  
4. **Include a `resetToken` field** for password‑reset flows, along with its own validation.  

Overall, `CustomerPassword` is concise, well‑structured, and follows best practices for DTO design in Java web applications. With the suggested small refinements, it would provide even stronger validation guarantees and better maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.customer;

import java.io.Serializable;
import javax.validation.constraints.Size;
import javax.validation.constraints.NotEmpty;
import com.salesmanager.shop.validation.FieldMatch;

@FieldMatch.List({
    @FieldMatch(first="password",second="checkPassword",message="password.notequal")
})
public class CustomerPassword implements Serializable {
	
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	@NotEmpty( message="{currentpassword.not.empty}")
	private String currentPassword;
	@Size(min=6, message="{newpassword.not.empty}")
	private String password;
	@Size(min=6, message="{repeatpassword.not.empty}")
	private String checkPassword;
	public String getCurrentPassword() {
		return currentPassword;
	}
	public void setCurrentPassword(String currentPassword) {
		this.currentPassword = currentPassword;
	}
	public String getPassword() {
		return password;
	}
	public void setPassword(String password) {
		this.password = password;
	}
	public String getCheckPassword() {
		return checkPassword;
	}
	public void setCheckPassword(String checkPassword) {
		this.checkPassword = checkPassword;
	}

}



```
