# PersistableCustomer.java

## Review

## 1. Summary  
**Purpose** – `PersistableCustomer` is a lightweight data transfer / persistence object that represents a customer entity within the *SalesManager* shop application. It extends `CustomerEntity` (presumably the core domain model) and adds fields that are only relevant when a customer is being created or updated (e.g. password handling, group membership, custom attributes).

**Key Components**  
| Component | Role |
|-----------|------|
| `password` / `repeatPassword` | Raw password fields used during registration or password‑reset flows. |
| `attributes` | List of `PersistableCustomerAttribute` – custom customer attributes that can be stored and retrieved. |
| `groups` | List of `PersistableGroup` – security groups to which the customer belongs. |
| Swagger annotations (`@ApiModel`, `@ApiModelProperty`) | Expose the model in generated API documentation. |

**Notable Design Patterns / Frameworks**  
- **DTO / Value Object** – The class acts as a data carrier between layers (e.g. REST controller ↔ persistence).  
- **Swagger/OpenAPI** – The annotations allow automatic API documentation generation.  
- **Java Persistence API (JPA) / Hibernate** – Although not directly visible, the naming (`Persistable…`) hints at mapping to a database entity.

---

## 2. Detailed Description  
`PersistableCustomer` builds upon the base domain `CustomerEntity` by adding transient fields that are needed during write operations but are not part of the core persistent state.  
During runtime:

1. **Construction** – The class is instantiated by a REST controller (or a service layer) when a customer is created/updated.  
2. **Population** – The controller maps incoming JSON to this object (via Jackson, for example).  
3. **Validation** – Basic validation (e.g. `password` not null) may be performed elsewhere; this class itself contains no business rules.  
4. **Persistence** – The service layer extracts the relevant parts (e.g. hashed password, groups, attributes) and stores them in the database.  
5. **Response** – After persistence, the service may return the same DTO (without the raw password fields) back to the client.

The class assumes that:
- The base `CustomerEntity` correctly defines the primary key, timestamps, etc.
- The caller will manage password hashing and validation; this DTO merely carries raw data.
- The lists `attributes` and `groups` will be non‑null or handled gracefully by the service layer.

---

## 3. Functions/Methods  
| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `setAttributes(List<PersistableCustomerAttribute>)` | Mutator for customer attributes. | List of attributes | void | Sets internal field. |
| `getAttributes()` | Accessor for customer attributes. | None | List | None |
| `getPassword()` | Accessor for raw password. | None | String | None |
| `setPassword(String)` | Mutator for raw password. | Password string | void | Sets internal field. |
| `getGroups()` | Accessor for security groups. | None | List | None |
| `setGroups(List<PersistableGroup>)` | Mutator for groups. | List of groups | void | Sets internal field. |
| `getRepeatPassword()` | Accessor for password confirmation. | None | String | None |
| `setRepeatPassword(String)` | Mutator for repeat password. | Password string | void | Sets internal field. |

*All methods are trivial getters/setters with no additional logic. The class is effectively a POJO.*

---

## 4. Dependencies  
| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `io.swagger.annotations.ApiModel` / `ApiModelProperty` | Third‑party | Swagger/OpenAPI annotations for API docs. |
| `java.util.List` | Standard Java | Collection for attributes and groups. |
| `com.salesmanager.shop.model.customer.attribute.PersistableCustomerAttribute` | Internal | Domain object representing a customer attribute. |
| `com.salesmanager.shop.model.security.PersistableGroup` | Internal | Domain object representing a security group. |
| `CustomerEntity` | Internal | Base class providing core customer fields. |

There are **no** heavy framework dependencies (e.g. no Spring, Hibernate annotations directly in this class). The class is largely framework‑agnostic but designed to integrate with a REST layer and a persistence layer.

---

## 5. Additional Notes  
### Strengths
- **Clarity & Simplicity** – The DTO is straightforward and easy to map to/from JSON.  
- **Swagger Integration** – Provides self‑documenting API models.  
- **Separation of Concerns** – Raw passwords are kept separate from the persisted customer entity, which is good practice.

### Potential Issues & Edge Cases  
1. **Password Handling** – Storing the raw password as a `String` in memory can be risky. Consider using `char[]` or a secure wrapper.  
2. **Null Lists** – `attributes` and `groups` may be `null`; callers must guard against `NullPointerException`. Initializing them to empty lists in the constructor could mitigate this.  
3. **Password Confirmation** – The class contains `repeatPassword`, but no logic checks that it matches `password`. Validation should be added at the service or controller layer.  
4. **Serialization Concerns** – `serialVersionUID` is present, but no `Serializable` interface is declared in this snippet. If serialization is required, ensure the class implements `Serializable`.  
5. **Equality & Hashing** – No `equals`/`hashCode` overrides. If instances are stored in collections that rely on these methods, the defaults from `Object` may be insufficient.  
6. **Immutability** – The DTO is mutable; for thread safety or functional style, consider making it immutable.  
7. **Lombok** – Repeated boilerplate getters/setters could be reduced using Lombok (`@Data`, `@Getter`, `@Setter`).  

### Future Enhancements  
- **Validation Annotations** – Add JSR‑380 (`@NotNull`, `@Size`, `@Pattern`) to enforce field constraints at the DTO level.  
- **Custom Annotations** – Create an annotation to automatically compare `password` and `repeatPassword`.  
- **Conversion Utility** – A method that converts this DTO to the core `CustomerEntity` (after hashing the password) could encapsulate mapping logic.  
- **Documentation** – Expand Swagger annotations on `attributes` and `groups` for clearer API docs.  
- **Security** – Implement masking of the password fields when the DTO is logged or serialized to prevent accidental leaks.

---

**Overall** – The class fulfills its role as a simple data carrier. With a few safety and validation improvements, it would be robust for production use in a typical Spring‑Boot + JPA + Swagger stack.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.customer;

import java.util.List;
import com.salesmanager.shop.model.customer.attribute.PersistableCustomerAttribute;
import com.salesmanager.shop.model.security.PersistableGroup;
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;



@ApiModel(value="Customer", description="Customer model object")
public class PersistableCustomer extends CustomerEntity {

	/**
	 * 
	 */
    @ApiModelProperty(notes = "Customer password")
	private String password = null;
    private String repeatPassword = null;
	private static final long serialVersionUID = 1L;
	private List<PersistableCustomerAttribute> attributes;
	private List<PersistableGroup> groups;
	
	
	public void setAttributes(List<PersistableCustomerAttribute> attributes) {
		this.attributes = attributes;
	}
	public List<PersistableCustomerAttribute> getAttributes() {
		return attributes;
	}

	public String getPassword() {
		return password;
	}
	public void setPassword(String password) {
		this.password = password;
	}
	public List<PersistableGroup> getGroups() {
		return groups;
	}
	public void setGroups(List<PersistableGroup> groups) {
		this.groups = groups;
	}
	public String getRepeatPassword() {
		return repeatPassword;
	}
	public void setRepeatPassword(String repeatPassword) {
		this.repeatPassword = repeatPassword;
	}
	

}



```
