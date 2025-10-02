# Address.java

## Review

## 1. Summary
The `Address` class is a plain Java object (POJO) that represents a customer’s billing or shipping address within the `com.salesmanager.shop.model.customer.address` package.  
It extends an (unknown) `AddressLocation` base class and implements `Serializable`, making it suitable for persistence or transport. The class primarily contains a collection of fields describing an address (name, company, phone, location details, coordinates, and billing flag) and the corresponding getters/setters. Swagger annotations (`@ApiModelProperty`) document the API model properties.  

Key components  
| Component | Purpose | Remarks |
|-----------|---------|---------|
| **Fields** | Store address data | Simple String/boolean fields; some redundancy (e.g., `stateProvince` + `zone`). |
| **Getters/Setters** | Encapsulation | No business logic, purely data access. |
| **Serializable** | Enable serialization | `serialVersionUID` defined. |
| **Swagger annotations** | API documentation | Provided on most fields, but validation annotations are commented out. |

The design is straightforward; it follows the JavaBeans convention and is likely used in data transfer objects (DTOs) for REST endpoints.

---

## 2. Detailed Description
### Class hierarchy
```text
AddressLocation   <--  Address
```
`AddressLocation` is not shown, but it presumably contains shared location fields (e.g., street, zip code, country). `Address` augments that with customer‑specific details such as name, company, and billing flag.

### Initialization
No explicit constructor is defined; Java will provide a default no‑arg constructor. The fields are initialized to `null` (or `false` for the boolean) until a setter is invoked or the object is deserialized.

### Runtime behavior
* The object acts purely as a data holder.  
* Setters allow mutation of any field.  
* Getters expose the stored values.  
* No validation, transformation, or business logic is present.

### Cleanup
There is no resource cleanup or finalization logic.

### Assumptions & Constraints
* The class assumes that callers will provide meaningful values.  
* Latitude/longitude are stored as `String`, which may lead to parsing errors downstream.  
* The class does not enforce any uniqueness or immutability guarantees.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `setStateProvince(String)` | Stores state/province name. | `stateProvince` | void | Mutates field |
| `setCountry(String)` | Stores 2‑letter country code. | `country` | void | Mutates field |
| `getCompany()` | Retrieve company name. | – | `String` | none |
| `setCompany(String)` | Set company name. | `company` | void | Mutates field |
| `getAddress()` | Retrieve street address. | – | `String` | none |
| `setAddress(String)` | Set street address. | `address` | void | Mutates field |
| `getCity()` | Retrieve city. | – | `String` | none |
| `setCity(String)` | Set city. | `city` | void | Mutates field |
| `getStateProvince()` | Retrieve state/province name. | – | `String` | none |
| `getCountry()` | Retrieve country code. | – | `String` | none |
| `setZone(String)` | Set 2‑letter state/province code. | `zone` | void | Mutates field |
| `getZone()` | Retrieve state/province code. | – | `String` | none |
| `setPhone(String)` | Set contact phone number. | `phone` | void | Mutates field |
| `getPhone()` | Retrieve phone number. | – | `String` | none |
| `getFirstName()` | Retrieve first name. | – | `String` | none |
| `setFirstName(String)` | Set first name. | `firstName` | void | Mutates field |
| `getLastName()` | Retrieve last name. | – | `String` | none |
| `setLastName(String)` | Set last name. | `lastName` | void | Mutates field |
| `isBillingAddress()` | Check if this is a billing address. | – | `boolean` | none |
| `setBillingAddress(boolean)` | Flag address type. | `billingAddress` | void | Mutates field |
| `getBilstateOther()` | Retrieve other state text (used when no code). | – | `String` | none |
| `setBilstateOther(String)` | Set other state text. | `bilstateOther` | void | Mutates field |
| `getLatitude()` | Retrieve latitude. | – | `String` | none |
| `setLatitude(String)` | Set latitude. | `latitude` | void | Mutates field |
| `getLongitude()` | Retrieve longitude. | – | `String` | none |
| `setLongitude(String)` | Set longitude. | `longitude` | void | Mutates field |

> **Reusable/Utility methods** – None. The class is purely a data holder.

---

## 4. Dependencies
| Dependency | Type | Usage |
|------------|------|-------|
| `io.swagger.annotations.ApiModelProperty` | Third‑party (Swagger/OpenAPI) | Document fields for API models. |
| `java.io.Serializable` | Standard JDK | Enable Java serialization. |
| `AddressLocation` | Local | Inherits location fields (implementation not shown). |

> No other external libraries are referenced. The class is platform‑agnostic, but assumes a runtime that supports Java serialization and the Swagger annotations (typically a Spring Boot or JAX‑RS application).

---

## 5. Additional Notes & Recommendations
### Strengths
* **Simplicity** – Easy to understand and maintain.  
* **Clear API documentation** – Swagger annotations help generate OpenAPI docs.  
* **Serializable** – Ready for session persistence or messaging.

### Weaknesses / Edge Cases
1. **No Validation**  
   - Validation annotations (`@NotEmpty`, `@Size`, `@Pattern`) are commented out. Without them, invalid or missing data can propagate silently.
2. **Redundant Fields**  
   - `stateProvince` (text) and `zone` (code) may both be needed, but the API may confuse clients about which to use.  
   - `bilstateOther` seems to be an “other” text for when `zone` is not available; its usage is unclear.
3. **Coordinate Representation**  
   - Latitude and longitude are stored as `String`. This makes numeric comparisons, validation, and geographic calculations error‑prone. `BigDecimal` or `double` would be safer.
4. **Missing `equals`, `hashCode`, `toString`**  
   - For value objects, these are essential for collections, logging, and debugging.
5. **Immutability**  
   - The class is fully mutable. If used in multi‑threaded contexts or cached DTOs, this could cause subtle bugs.
6. **JavaBean Conventions**  
   - The `isBillingAddress` method follows the standard boolean getter naming. Good.

### Potential Enhancements
| Area | Suggested Improvement |
|------|------------------------|
| **Validation** | Un‑comment and apply Hibernate Validator annotations (e.g., `@NotBlank`, `@Pattern`) or use custom validation logic in service layer. |
| **Coordinate Types** | Change `latitude`/`longitude` to `BigDecimal` or `Double` with proper validation (range checks). |
| **Value Object Pattern** | Make the class immutable (`final` fields, constructor-only). Add `equals`, `hashCode`, `toString`. |
| **Reduce Boilerplate** | Use Lombok (`@Data`, `@Builder`, `@AllArgsConstructor`, etc.) to auto‑generate getters/setters, constructors, and other methods. |
| **Documentation** | Add Javadoc to each field/method explaining business semantics. |
| **Clarify State/Zone Fields** | Consolidate or clearly document when each field should be used; perhaps replace `bilstateOther` with a `stateDescription` that is used only when `zone` is null. |
| **Unit Tests** | Write tests for any future business logic, e.g., validation or formatting. |
| **DTO vs Entity** | If this class is used as a DTO, keep it lightweight. If it becomes an entity, map it to a JPA table and add appropriate annotations. |

### Final Verdict
The `Address` class is a fine starting point for representing customer addresses in a REST API. However, to make it production‑ready, consider adding validation, proper type handling for coordinates, and value‑object semantics. These changes will improve robustness, maintainability, and clarity for API consumers.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.customer.address;

import java.io.Serializable;

import io.swagger.annotations.ApiModelProperty;


/**
 * Customer or someone address
 * @author carlsamson
 *
 */
public class Address extends AddressLocation implements Serializable {
	
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;

	@ApiModelProperty(notes = "Customer billing first name")
	//@NotEmpty(message="{NotEmpty.customer.firstName}")
	private String firstName;
	
	@ApiModelProperty(notes = "Customer billing last name")
	//@NotEmpty(message="{NotEmpty.customer.lastName}")
	private String lastName;
	
	private String bilstateOther;

	private String company;

	private String phone;
	@ApiModelProperty(notes = "Customer billing or shipping address")
	private String address;
	@ApiModelProperty(notes = "Customer billing or shipping city")
	private String city;
	

	
	@ApiModelProperty(notes = "Customer billing or shipping state / province (if no 2 letter codes, example: North estate)")
	private String stateProvince;
	private boolean billingAddress;
	
	private String latitude;
	private String longitude;
	
	@ApiModelProperty(notes = "Customer billing or shipping state / province (2 letter code CA, ON...)")
	private String zone;//code
	
	@ApiModelProperty(notes = "Customer billing or shipping country code (2 letter code US, CA, UK, IT, IN, CN...)")
	//@NotEmpty(message="{NotEmpty.customer.billing.country}")
	private String country;//code
	


	public void setStateProvince(String stateProvince) {
		this.stateProvince = stateProvince;
	}

	public void setCountry(String country) {
		this.country = country;
	}



	public String getCompany() {
		return company;
	}

	public void setCompany(String company) {
		this.company = company;
	}

	public String getAddress() {
		return address;
	}

	public void setAddress(String address) {
		this.address = address;
	}

	public String getCity() {
		return city;
	}

	public void setCity(String city) {
		this.city = city;
	}



	public String getStateProvince() {
		return stateProvince;
	}

	public String getCountry() {
		return country;
	}

	public void setZone(String zone) {
		this.zone = zone;
	}

	public String getZone() {
		return zone;
	}

	public void setPhone(String phone) {
		this.phone = phone;
	}

	public String getPhone() {
		return phone;
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

   public boolean isBillingAddress()
    {
        return billingAddress;
    }

    public void setBillingAddress( boolean billingAddress )
    {
        this.billingAddress = billingAddress;
    }

    public String getBilstateOther()
    {
        return bilstateOther;
    }

    public void setBilstateOther( String bilstateOther )
    {
        this.bilstateOther = bilstateOther;
    }

	public String getLatitude() {
		return latitude;
	}

	public void setLatitude(String latitude) {
		this.latitude = latitude;
	}

	public String getLongitude() {
		return longitude;
	}

	public void setLongitude(String longitude) {
		this.longitude = longitude;
	}

}



```
