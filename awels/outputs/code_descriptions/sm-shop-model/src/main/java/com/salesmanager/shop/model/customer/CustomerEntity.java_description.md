# CustomerEntity.java

## Review

## 1. Summary  

**Purpose & Functionality**  
`CustomerEntity` is a DTO (Data‑Transfer Object) that represents a customer within the *Sales Manager* shop domain. It extends a base `Customer` class (not shown) and enriches it with fields required for customer registration, profile management, and social‑login integration. The class is designed to be serializable, validated via JSR‑303/JSR‑380 annotations, and documented for Swagger.

**Key Components**  
| Component | Role |
|-----------|------|
| `emailAddress` | Unique identifier for login and contact; validated as an email. |
| `billing` / `delivery` | Address objects for the customer’s billing and delivery addresses. |
| `gender`, `language` | Demographic metadata. |
| `firstName`, `lastName` | Personal names. |
| `provider` | Indicates third‑party authentication provider (e.g., “facebook”). |
| `storeCode` | Associates the customer with a particular store instance. |
| `userName` | Optional login alias (can be the email). |
| `rating`, `ratingCount` | Customer rating metrics. |
| Validation & Swagger annotations | Ensure correct data and provide API documentation. |

**Design Patterns & Frameworks**  
- **Builder/DTO Pattern**: The class is purely a data holder with getters/setters.  
- **Validation**: Uses JSR‑303 (`@NotEmpty`, `@Email`, `@Valid`).  
- **Swagger**: `@ApiModelProperty` annotations expose model metadata.  
- **Spring**: `org.springframework.validation.annotation.Validated` hints at integration in Spring MVC/REST controllers.

---

## 2. Detailed Description  

### Core Structure  
`CustomerEntity` inherits from `Customer`, which presumably contains shared attributes (e.g., `id`, `createdAt`). The subclass adds customer‑specific attributes needed for the shop’s API surface.  

### Execution Flow  
1. **Instantiation** – The object is typically created by a controller or service layer.  
2. **Population** – JSON/XML payload is deserialized into this class.  
3. **Validation** – Spring’s validator (or any JSR‑380 provider) checks:
   - `emailAddress` is a non‑empty valid email.  
   - `billing` and `delivery` addresses are valid (due to `@Valid`).  
4. **Business Logic** – Downstream services (e.g., registration, profile update) operate on the populated fields.  
5. **Persistence** – The entity might be mapped to a database table (though mapping annotations are absent, implying separate persistence entity).  
6. **Cleanup** – As a plain DTO, no explicit cleanup is required.  

### Assumptions & Constraints  
- `emailAddress` must be unique; however, uniqueness is not enforced here—assumed at database or service level.  
- `gender` accepts only `"M"` or `"F"`; no validation is implemented (could be extended).  
- `language` expects a two‑letter ISO code; no format enforcement here.  
- `rating` and `ratingCount` default to 0; no concurrency control if updated concurrently.  

### Architecture & Design Choices  
- **Separation of Concerns**: Keeps validation logic close to data definitions, enabling reuse in REST APIs.  
- **Extensibility**: Additional provider fields (e.g., `oauthToken`) can be added without altering existing methods.  
- **Serialization**: Implements `Serializable` for potential caching or remote transfer.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `setUserName(String)` | Stores login alias. | `userName` | void | Assigns to field. |
| `getUserName()` | Retrieves login alias. | – | `String` | – |
| `setStoreCode(String)` | Associates customer with a store. | `storeCode` | void | Assigns to field. |
| `getStoreCode()` | Retrieves store code. | – | `String` | – |
| `setEmailAddress(String)` | Stores email; required for registration. | `emailAddress` | void | Assigns to field. |
| `getEmailAddress()` | Retrieves email. | – | `String` | – |
| `setLanguage(String)` | Sets customer language. | `language` | void | Assigns to field. |
| `getLanguage()` | Retrieves language. | – | `String` | – |
| `getBilling()` | Gets billing address. | – | `Address` | – |
| `setBilling(Address)` | Sets billing address. | `billing` | void | Assigns to field. |
| `getDelivery()` | Gets delivery address. | – | `Address` | – |
| `setDelivery(Address)` | Sets delivery address. | `delivery` | void | Assigns to field. |
| `setGender(String)` | Stores gender. | `gender` | void | Assigns to field. |
| `getGender()` | Retrieves gender. | – | `String` | – |
| `getFirstName()` | Retrieves first name. | – | `String` | – |
| `setFirstName(String)` | Stores first name. | `firstName` | void | Assigns to field. |
| `getLastName()` | Retrieves last name. | – | `String` | – |
| `setLastName(String)` | Stores last name. | `lastName` | void | Assigns to field. |
| `getRatingCount()` | Retrieves number of ratings. | – | `int` | – |
| `setRatingCount(int)` | Stores rating count. | `ratingCount` | void | Assigns to field. |
| `getRating()` | Retrieves rating average. | – | `Double` | – |
| `setRating(Double)` | Stores rating value. | `rating` | void | Assigns to field. |
| `getProvider()` | Retrieves auth provider. | – | `String` | – |
| `setProvider(String)` | Stores provider. | `provider` | void | Assigns to field. |

All methods are simple property accessors; no complex logic is embedded.

---

## 4. Dependencies  

| Library / Framework | Usage |
|---------------------|-------|
| **javax.validation** (`@NotEmpty`, `@Email`, `@Valid`) | Bean Validation API for field constraints. |
| **org.springframework.validation.annotation.Validated** | Marks the class for Spring’s validation framework. |
| **io.swagger.annotations.ApiModelProperty** | Provides Swagger/OpenAPI documentation for each field. |
| **com.salesmanager.shop.model.customer.address.Address** | Custom address type used for billing/delivery. |
| **java.io.Serializable** | Enables Java serialization. |

All dependencies are standard or well‑established third‑party libraries. No platform‑specific assumptions beyond the typical Java EE / Spring stack.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  
- **Gender & Language Validation**: No enforcement of allowed values; invalid strings could silently persist. Adding `@Pattern` or an enum would improve robustness.  
- **Email Uniqueness**: The annotation ensures format but not uniqueness; enforce at the database or service layer.  
- **Null Addresses**: `billing` and `delivery` are annotated with `@Valid`, but the fields themselves are nullable. If `@NotNull` is required, add that annotation.  
- **Thread‑Safety**: The class is mutable; concurrent updates should be coordinated externally.  
- **Serialization**: The class relies on default serialization. If the application uses JSON, consider `Jackson` annotations for naming strategies.

### Suggested Enhancements  
1. **Immutable DTO** – Switch to a builder pattern or make fields final to avoid accidental mutation.  
2. **Enum Types** – Replace `gender` and `language` with enums to enforce valid values.  
3. **Custom Validation** – Implement a validator for `storeCode` or `provider` to match expected patterns.  
4. **Swagger Enhancements** – Add `required = true` to mandatory fields in `@ApiModelProperty`.  
5. **Documentation** – Provide Javadoc for each method and class, clarifying usage and constraints.  
6. **Unit Tests** – Add tests for validation constraints to guarantee API contract.  

Overall, the class is clean, well‑documented, and fits its intended role as a data holder for customer information in a Spring‑based e‑commerce application.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.customer;

import java.io.Serializable;

import javax.validation.Valid;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotEmpty;

import org.springframework.validation.annotation.Validated;

import com.salesmanager.shop.model.customer.address.Address;

import io.swagger.annotations.ApiModelProperty;

public class CustomerEntity extends Customer implements Serializable {

	/**
	 *
	 */
	private static final long serialVersionUID = 1L;

	@ApiModelProperty(notes = "Customer email address. Required for registration")
	@Email (message="{messages.invalid.email}")
    @NotEmpty(message="{NotEmpty.customer.emailAddress}")
	private String emailAddress;
	@Valid
	@ApiModelProperty(notes = "Customer billing address")
	private Address billing;
	private Address delivery;
	@ApiModelProperty(notes = "Customer gender M | F")
	private String gender;

	@ApiModelProperty(notes = "2 letters language code en | fr | ...")
	private String language;
	private String firstName;
	private String lastName;
	
	private String provider;//online, facebook ...

	
	private String storeCode;
	
	//@ApiModelProperty(notes = "Username (use email address)")
	//@NotEmpty(message="{NotEmpty.customer.userName}")
	//can be email or anything else
	private String userName;
	
	private Double rating = 0D;
	private int ratingCount;
	
	public void setUserName(final String userName) {
		this.userName = userName;
	}

	public String getUserName() {
		return userName;
	}


	public void setStoreCode(final String storeCode) {
		this.storeCode = storeCode;
	}


	public String getStoreCode() {
		return storeCode;
	}


	public void setEmailAddress(final String emailAddress) {
		this.emailAddress = emailAddress;
	}
	

	public String getEmailAddress() {
		return emailAddress;
	}


	public void setLanguage(final String language) {
		this.language = language;
	}
	public String getLanguage() {
		return language;
	}
	

	public Address getBilling() {
		return billing;
	}
	public void setBilling(final Address billing) {
		this.billing = billing;
	}
	public Address getDelivery() {
		return delivery;
	}
	public void setDelivery(final Address delivery) {
		this.delivery = delivery;
	}
	public void setGender(final String gender) {
		this.gender = gender;
	}
	public String getGender() {
		return gender;
	}


	public String getFirstName() {
		return firstName;
	}


	public void setFirstName(String firstName) {
		this.firstName = firstName;
	}


	public String getLastName() {
		return lastName;
	}


	public void setLastName(String lastName) {
		this.lastName = lastName;
	}


	public int getRatingCount() {
		return ratingCount;
	}

	public void setRatingCount(int ratingCount) {
		this.ratingCount = ratingCount;
	}

	public Double getRating() {
		return rating;
	}

	public void setRating(Double rating) {
		this.rating = rating;
	}

	public String getProvider() {
		return provider;
	}

	public void setProvider(String provider) {
		this.provider = provider;
	}



    

}



```
