# AbstractUserConnection.java

## Review

## 1. Summary  
The file defines an abstract JPA *mapped superclass* that represents a generic remote user connection (e.g., a social‑login link).  
* **Purpose** – Provides the common persistence mapping and behaviour for concrete connection entities (e.g., `FacebookUserConnection`, `GoogleUserConnection`).  
* **Key Components**  
  * A handful of OAuth‑style fields (`accessToken`, `refreshToken`, `secret`, etc.).  
  * Abstract methods that subclasses must implement: provider identifiers, user identifiers and a generic `getId()` hook.  
  * Implements `RemoteUser` (interface not shown) and `Serializable`.  
* **Notable Design Choices**  
  * Uses **JPA `@MappedSuperclass`** to share column mapping across subclasses.  
  * Declares itself **`@Deprecated`**, signalling that a newer design is likely in use.  
  * Generic type parameter `<P>` is used only in the protected `getId()` abstract method; the concrete ID type is delegated to subclasses.

---

## 2. Detailed Description  
### Architecture & Flow  
1. **Initialization** – JPA will create an instance of a concrete subclass (e.g., `FacebookUserConnection`). The superclass provides a no‑argument default constructor (implicit because none is declared).  
2. **Runtime** –  
   * The framework (Spring Data/JPA) persists instances; all fields in this class become columns in the subclass’s table.  
   * Getters/setters expose the OAuth token, display name, avatar URL, rank, etc.  
   * Subclasses supply provider‑specific logic through the abstract methods (`getProviderId()`, `getProviderUserId()`, etc.).  
3. **Cleanup** – Not applicable; the class is a plain data holder.

### Assumptions & Constraints  
* **Persistence** – Assumes an ORM environment that recognises `@MappedSuperclass`.  
* **Serialization** – Implements `Serializable` mainly for framework‑level caching or session replication.  
* **Provider Integration** – The `RemoteUser` interface probably defines a contract for remote authentication; the subclass must supply the missing details.  
* **Null Handling** – No defensive checks; fields may be `null` by default.  

### Design Choices  
* **Generic ID (`<P>`)** – Intended to allow subclasses to expose a strongly‑typed primary key. However, the class does not itself declare an `@Id` field, so the key must be declared in the subclass.  
* **Primitive `int` for `userRank`** – Might be better as `Integer` to allow `null` (unknown rank).  
* **Deprecated Annotation** – Signals that this approach is legacy; a new mechanism likely exists.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getAccessToken()` | Return OAuth access token. | – | `String` | – |
| `setAccessToken(String)` | Set OAuth access token. | `String` | – | Updates field |
| `getDisplayName()` | Return user’s display name. | – | `String` | – |
| `setDisplayName(String)` | Set display name. | `String` | – | – |
| `getExpireTime()` | Return token expiry timestamp. | – | `Long` | – |
| `setExpireTime(Long)` | Set expiry time. | `Long` | – | – |
| `getImageUrl()` | Return avatar URL. | – | `String` | – |
| `setImageUrl(String)` | Set avatar URL. | `String` | – | – |
| `getProfileUrl()` | Return profile page URL. | – | `String` | – |
| `setProfileUrl(String)` | Set profile URL. | `String` | – | – |
| `getRank()` | Return user rank. | – | `int` | – |
| `setRank(int)` | Set rank. | `int` | – | – |
| `getRefreshToken()` | Return OAuth refresh token. | – | `String` | – |
| `setRefreshToken(String)` | Set refresh token. | `String` | – | – |
| `getSecret()` | Return OAuth secret (e.g., OAuth 1.0). | – | `String` | – |
| `setSecret(String)` | Set secret. | `String` | – | – |
| `getProviderId()` *(abstract)* | Return provider id (e.g., `"facebook"`). | – | `String` | – |
| `setProviderId(String)` *(abstract)* | Set provider id. | `String` | – | – |
| `getProviderUserId()` *(abstract)* | Return provider‑specific user id. | – | `String` | – |
| `setProviderUserId(String)` *(abstract)* | Set provider‑specific user id. | `String` | – | – |
| `getUserId()` *(abstract)* | Return internal application user id. | – | `String` | – |
| `setUserId(String)` *(abstract)* | Set internal user id. | `String` | – | – |
| `getId()` *(protected abstract)* | Return subclass‑defined entity id. | – | `P` | – |

*All abstract methods must be implemented by concrete subclasses, typically mapping to JPA `@Id` fields and provider‑specific identifiers.*

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.MappedSuperclass` | **Third‑party** (JPA API) | Requires a JPA implementation (e.g., Hibernate, EclipseLink). |
| `java.io.Serializable` | **Standard** | Enables Java serialization; primarily for ORM or HTTP session storage. |
| `RemoteUser` | **Project‑specific** | Interface not provided; assumed to define basic remote‑user contract. |
| `@Deprecated` | **Standard** | Annotation for deprecation warnings. |

No external libraries (e.g., Lombok, Jackson) are used.

---

## 5. Additional Notes  

### Strengths  
* Clear separation of common persistence fields from provider‑specific logic.  
* Uses JPA’s `@MappedSuperclass` to avoid field duplication across entities.  
* Implements `Serializable`, useful for caching/session replication.  

### Potential Issues & Edge Cases  
1. **Missing No‑arg Constructor** – JPA requires a protected or public no‑argument constructor; it’s implicitly present but can be made explicit for readability.  
2. **`@Deprecated` Flag** – The class is marked deprecated but remains in the code base; any new development should avoid extending it.  
3. **Equality & Hashing** – No `equals()`/`hashCode()` overrides. For persistence and collection usage, these should be based on the entity’s identity (`id` or a combination of providerId & providerUserId).  
4. **Thread Safety** – The class is mutable; concurrent access to the same instance could cause race conditions. Not an issue in typical JPA usage, but worth documenting.  
5. **Null Handling** – Setters do not validate input; callers must ensure non‑null values where required.  
6. **Generic `P` Only in `getId()`** – Since the ID type is only exposed via a protected abstract method, the benefit of generics is limited. A more common pattern is to declare the `@Id` field in this superclass with a generic type.  
7. **Primitive `int` for Rank** – If rank is optional, consider using `Integer` or a default sentinel value.  
8. **Security** – Sensitive data (tokens, secrets) are stored in plain fields; ensure that the persistence layer encrypts them or that the application handles them securely.  

### Future Enhancements  
* **Add `equals()`, `hashCode()`, and `toString()`** – Improve debugging and collection behaviour.  
* **Define `@Id` in the superclass** – Simplify ID handling and reduce duplication in subclasses.  
* **Use Lombok** – If the project allows, reduce boilerplate.  
* **Validation Annotations** – Add `@NotNull`, `@Size`, etc., for Bean Validation.  
* **Remove or Replace Deprecated Code** – Provide migration guidance or a concrete replacement class.  
* **Encryption Support** – Integrate attribute converters to encrypt/decrypt tokens in the database.  

Overall, the class serves as a simple persistence‑ready template for remote user connections, but its deprecation and lack of essential methods (constructor, equality, validation) limit its immediate usefulness in new code.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.customer.connection;

import java.io.Serializable;

import javax.persistence.MappedSuperclass;


@Deprecated
@MappedSuperclass
public abstract class AbstractUserConnection<P> implements RemoteUser,
		Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;

	private String accessToken;
	private String displayName;
	private Long expireTime;
	private String imageUrl;
	private String profileUrl;
	private int userRank;
	private String refreshToken;
	private String secret;


	public String getAccessToken() {
		return accessToken;
	}

	public void setAccessToken(String accessToken) {
		this.accessToken = accessToken;
	}

	public String getDisplayName() {
		return displayName;
	}

	public void setDisplayName(String displayName) {
		this.displayName = displayName;
	}

	public Long getExpireTime() {
		return expireTime;
	}

	public void setExpireTime(Long expireTime) {
		this.expireTime = expireTime;
	}

	public String getImageUrl() {
		return imageUrl;
	}

	public void setImageUrl(String imageUrl) {
		this.imageUrl = imageUrl;
	}

	public String getProfileUrl() {
		return profileUrl;
	}

	public void setProfileUrl(String profileUrl) {
		this.profileUrl = profileUrl;
	}

	public abstract String getProviderId();

	public abstract void setProviderId(String providerId);

	public abstract String getProviderUserId();

	public abstract void setProviderUserId(String providerUserId);

	public int getRank() {
		return userRank;
	}

	public void setRank(int userRank) {
		this.userRank = userRank;
	}

	public String getRefreshToken() {
		return refreshToken;
	}

	public void setRefreshToken(String refreshToken) {
		this.refreshToken = refreshToken;
	}

	public String getSecret() {
		return secret;
	}

	public void setSecret(String secret) {
		this.secret = secret;
	}

	public abstract String getUserId();

	public abstract void setUserId(String userId);

	protected abstract P getId();
}



```
