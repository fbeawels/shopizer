# OrderTotalInputParameters.java

## Review

## 1. Summary  
**Purpose**  
`OrderTotalInputParameters` is a simple Java bean that bundles together all the data required by the order‑total calculation engine (referred to as “drules engine” in the comment). The engine mutates two of the fields (`discount` and `totalCode`) and reads the rest to perform its calculations.  

**Key Components**  
- **Input fields** (`productId`, `itemManufacturerCode`, `itemCategoryCode`, `shippingMethod`, `promoCode`, `date`) – values supplied by the calling code (e.g., order service).  
- **Output fields** (`discount`, `totalCode`) – set by the engine to return the calculated discount and a code identifying the applied rule.  
- **Getters/Setters** – standard JavaBean accessors to allow the engine to read/write the properties.  

**Design Patterns & Libraries**  
- Classic **JavaBean** pattern – plain POJO with private fields and public getters/setters.  
- No external libraries or frameworks are used; only `java.util.Date`.  

---

## 2. Detailed Description  
1. **Construction & Initialization**  
   - No explicit constructor is declared; the default no‑args constructor is provided by the compiler.  
   - `date` is initialized to the current system time at object creation. This means every instance records the creation timestamp unless overridden.

2. **Runtime Behavior**  
   - The client (e.g., an order service or controller) creates an instance, populates the input fields, and hands it to the engine.  
   - The engine calls the getters to fetch inputs and then sets the `discount` and `totalCode` via the setters.  
   - After engine execution, the client can retrieve the computed values through the same getters.

3. **Assumptions & Constraints**  
   - **Immutability**: The bean is mutable; external code may change any field after engine execution, potentially invalidating results.  
   - **Thread‑safety**: No synchronization is provided; the object is **not** thread‑safe.  
   - **Date semantics**: `java.util.Date` is mutable; callers can alter the returned date if they keep a reference.  
   - **Null handling**: All fields are nullable except the primitive `productId`. The engine must defensively check for `null` before using string fields.  

4. **Architecture & Design Choices**  
   - Keeping the bean separate from the engine decouples calculation logic from data representation.  
   - Using a mutable bean for I/O is typical in legacy Java systems; newer designs might prefer immutable value objects or builder patterns.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `Double getDiscount()` | Retrieve the calculated discount. | – | `Double` | – |
| `void setDiscount(Double discount)` | Set the calculated discount. | `Double discount` | – | updates internal state |
| `String getTotalCode()` | Retrieve the code of the applied total rule. | – | `String` | – |
| `void setTotalCode(String totalCode)` | Set the rule code. | `String totalCode` | – | updates internal state |
| `String getItemManufacturerCode()` | Getter for manufacturer code. | – | `String` | – |
| `void setItemManufacturerCode(String itemManufacturerCode)` | Setter. | `String` | – | updates internal state |
| `String getItemCategoryCode()` | Getter. | – | `String` | – |
| `void setItemCategoryCode(String itemCategoryCode)` | Setter. | `String` | – | updates internal state |
| `long getProductId()` | Getter for product id. | – | `long` | – |
| `void setProductId(long productId)` | Setter. | `long` | – | updates internal state |
| `String getShippingMethod()` | Getter. | – | `String` | – |
| `void setShippingMethod(String shippingMethod)` | Setter. | `String` | – | updates internal state |
| `String getPromoCode()` | Getter. | – | `String` | – |
| `void setPromoCode(String promoCode)` | Setter. | `String` | – | updates internal state |
| `Date getDate()` | Return the timestamp when the bean was created (or set). | – | `Date` | – (returns reference) |
| `void setDate(Date date)` | Override the timestamp. | `Date` | – | updates internal state |

> **Utility note** – All setters are simple assignments; no validation is performed. If the engine requires stricter constraints (e.g., non‑null codes), validation should be added either here or in the calling code.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.util.Date` | Standard JDK | Mutable; consider `java.time.Instant` or `LocalDateTime` in Java 8+ for better semantics. |
| None other | | The class does not depend on any third‑party libraries or frameworks. |

---

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Issues  
1. **Mutable `Date` Exposure** – Returning the internal `Date` instance allows callers to mutate the bean’s timestamp. Wrap the date in an immutable copy (`new Date(date.getTime())`) in the getter.  
2. **Null Values** – The engine must handle `null` strings; otherwise, a `NullPointerException` may arise. Adding simple null‑checks or default values could improve robustness.  
3. **Thread‑Safety** – If the bean is shared across threads (unlikely but possible in a multi‑threaded engine), consider making it immutable or synchronizing access.  
4. **Legacy Design** – Modern Java prefers immutable value objects; a builder pattern or constructor‑based injection could reduce accidental mutation.  
5. **Field Naming** – The class uses “ItemManufacturerCode” but getter/setter names follow the standard pattern. Ensure consistency with the rest of the codebase.

### Future Enhancements  
- **Immutability**: Replace setters with constructor arguments or a builder, making the bean immutable after creation.  
- **Type Safety**: Replace raw `String` codes with enums (e.g., `ShippingMethod`, `PromoCode`) to avoid typos.  
- **Date Handling**: Adopt `java.time` API (`Instant`, `ZonedDateTime`) for clearer semantics and better timezone handling.  
- **Validation Layer**: Introduce a validation method or use frameworks like Hibernate Validator (`@NotNull`, `@Pattern`) if the bean is used with frameworks that support bean validation.  
- **Documentation**: Add Javadoc for each method explaining whether the field is input or output and any constraints.  

### Suggested Refactor (Minimal)  

```java
public final class OrderTotalInputParameters {

    private final Double discount;
    private final String totalCode;
    private final long productId;
    private final String itemManufacturerCode;
    private final String itemCategoryCode;
    private final String shippingMethod;
    private final String promoCode;
    private final Instant date; // java.time

    public OrderTotalInputParameters(long productId, String itemManufacturerCode,
                                     String itemCategoryCode, String shippingMethod,
                                     String promoCode, Instant date) {
        this.productId = productId;
        this.itemManufacturerCode = itemManufacturerCode;
        this.itemCategoryCode = itemCategoryCode;
        this.shippingMethod = shippingMethod;
        this.promoCode = promoCode;
        this.date = date != null ? date : Instant.now();
        this.discount = null;
        this.totalCode = null;
    }

    // getters only, no setters
}
```

This approach ensures the bean is immutable, safe, and easier to reason about.

---

**Overall**  
The class is straightforward and serves its purpose as a data carrier for the order‑total engine. While functional, it follows a legacy mutable pattern that can be modernized for better safety and maintainability. Implementing the above suggestions would make the codebase more robust and future‑proof.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.order.total;

import java.util.Date;

/**
 * Input itm to be added to drules engine
 * @author carlsamson
 *
 */
public class OrderTotalInputParameters {
	
	private Double discount;//output value set by engine
	private String totalCode;//output value set by engine
	
	//input parameters
	private long productId;
	private String itemManufacturerCode;
	private String itemCategoryCode;
	private String shippingMethod;
	private String promoCode;
	private Date date = new Date();
	
	//might add variation based on other objects such as Customer
	
	public Double getDiscount() {
		return discount;
	}
	public void setDiscount(Double discount) {
		this.discount = discount;
	}
	public String getTotalCode() {
		return totalCode;
	}
	public void setTotalCode(String totalCode) {
		this.totalCode = totalCode;
	}
	public String getItemManufacturerCode() {
		return itemManufacturerCode;
	}
	public void setItemManufacturerCode(String itemManufacturerCode) {
		this.itemManufacturerCode = itemManufacturerCode;
	}
	public String getItemCategoryCode() {
		return itemCategoryCode;
	}
	public void setItemCategoryCode(String itemCategoryCode) {
		this.itemCategoryCode = itemCategoryCode;
	}
	public long getProductId() {
		return productId;
	}
	public void setProductId(long productId) {
		this.productId = productId;
	}
	public String getShippingMethod() {
		return shippingMethod;
	}
	public void setShippingMethod(String shippingMethod) {
		this.shippingMethod = shippingMethod;
	}
	public String getPromoCode() {
		return promoCode;
	}
	public void setPromoCode(String promoCode) {
		this.promoCode = promoCode;
	}
	public Date getDate() {
		return date;
	}
	public void setDate(Date date) {
		this.date = date;
	}

	


}



```
