# ReadableShoppingCartItem.java

## Review

## 1. Summary  
`ReadableShoppingCartItem` is a simple Java POJO that represents a single line‑item in a shopping‑cart.  
It extends `ReadableMinimalProduct`, thereby inheriting the basic product metadata (name, id, price, etc.) and adds cart‑specific attributes such as the quantity‑derived subtotal, a formatted display string, selected attributes, and two product‑variation objects (`variant` and `variantValue`).  

The class is purely data‑centric – it contains no business logic, validation or transformation – and is intended for transfer between the service layer and the presentation layer (e.g., JSON serialization for a REST API).

---

## 2. Detailed Description  

### Core Components  
| Field | Type | Purpose |
|-------|------|---------|
| `subTotal` | `BigDecimal` | Raw numeric subtotal for this cart line (price × quantity). |
| `displaySubTotal` | `String` | Formatted currency string, e.g. “$19.99”. |
| `cartItemattributes` | `List<ReadableShoppingCartAttribute>` | Collection of chosen attributes (colour, size, etc.). |
| `variant` | `ReadableProductVariation` | Product variation (e.g. SKU). |
| `variantValue` | `ReadableProductVariation` | The value of the selected variation attribute. |

The class inherits all the fields from `ReadableMinimalProduct` (product id, name, image URLs, base price, etc.) which allows it to be used wherever a “read‑only” product representation is needed.

### Execution Flow  
1. **Construction** – The default constructor (implicitly provided) creates an empty instance.  
2. **Population** – Service code populates the fields using setters or by copying from an entity.  
3. **Serialization** – The object is typically serialized (e.g., Jackson) to JSON for a REST response.  
4. **Deserialization** – For inbound requests a similar immutable structure may be used, but this class itself is not designed for inbound mutation.

### Assumptions & Constraints  
- The class is **mutable**; all fields can be changed after construction.  
- It relies on external POJOs (`ReadableMinimalProduct`, `ReadableShoppingCartAttribute`, `ReadableProductVariation`) that are expected to be serializable.  
- No validation logic is present; callers must ensure values are non‑null where required.  
- The class is **serializable** (implements `Serializable`) but does not override `readObject`/`writeObject`.  
- There are no `equals()`, `hashCode()`, or `toString()` implementations – default object identity semantics apply.

### Design Choices  
- **Inheritance** over composition: extending `ReadableMinimalProduct` keeps the API flat for consumer code but introduces tight coupling and a fragile “is‑a” relationship.  
- **Raw `BigDecimal`** for money instead of a dedicated money type (e.g., JSR‑354 `Money`) keeps the model simple but sacrifices locale‑aware formatting.  
- The use of a separate `displaySubTotal` string allows the server to off‑load currency formatting, but it duplicates the value and can become stale if `subTotal` changes.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Notes |
|--------|-----------|---------|-------|
| `getSubTotal()` | `BigDecimal getSubTotal()` | Returns the numeric subtotal. | |
| `setSubTotal(BigDecimal)` | `void setSubTotal(BigDecimal)` | Sets the numeric subtotal. | No validation of non‑negative value. |
| `getDisplaySubTotal()` | `String getDisplaySubTotal()` | Returns the formatted subtotal. | |
| `setDisplaySubTotal(String)` | `void setDisplaySubTotal(String)` | Sets the formatted subtotal. | Must match `subTotal` for consistency. |
| `getCartItemattributes()` | `List<ReadableShoppingCartAttribute> getCartItemattributes()` | Returns list of selected attributes. | Returns the internal mutable list; callers can modify it directly. |
| `setCartItemattributes(List<ReadableShoppingCartAttribute>)` | `void setCartItemattributes(List<ReadableShoppingCartAttribute>)` | Sets the list of attributes. | No defensive copy. |
| `getVariant()` | `ReadableProductVariation getVariant()` | Returns the product variation. | |
| `setVariant(ReadableProductVariation)` | `void setVariant(ReadableProductVariation)` | Sets the product variation. | |
| `getVariantValue()` | `ReadableProductVariation getVariantValue()` | Returns the variation value. | |
| `setVariantValue(ReadableProductVariation)` | `void setVariantValue(ReadableProductVariation)` | Sets the variation value. | |

> **Reusable/Utility Methods** – None beyond trivial getters/setters. The class is essentially a data container.

---

## 4. Dependencies  

| Library/Package | Role | Standard / Third‑Party |
|-----------------|------|------------------------|
| `java.io.Serializable` | Enables Java serialization. | Standard |
| `java.math.BigDecimal` | Precise decimal arithmetic for money. | Standard |
| `java.util.ArrayList` / `java.util.List` | Collection framework. | Standard |
| `com.salesmanager.shop.model.catalog.product.ReadableMinimalProduct` | Base product representation. | Project‑specific |
| `com.salesmanager.shop.model.catalog.product.variation.ReadableProductVariation` | Product variation representation. | Project‑specific |
| `com.salesmanager.shop.model.shoppingcart.ReadableShoppingCartAttribute` | Attribute key/value representation. | Project‑specific |

No external frameworks (e.g., Jackson, Lombok) are directly referenced in the code; however, the class is intended to be serializable by typical JSON libraries.

---

## 5. Additional Notes & Recommendations  

### Code‑Quality & Maintenance  
- **Immutability** – Consider making the class immutable (final fields, no setters, defensive copies). This reduces accidental mutation and simplifies thread‑safety for API responses.  
- **Defensive Copies** – `getCartItemattributes()` currently exposes the internal list. If immutability is not adopted, return `Collections.unmodifiableList(...)` or a defensive copy.  
- **Validation** – Add simple validation (e.g., non‑null `subTotal`, non‑negative values) or use annotations (`@NotNull`, `@DecimalMin("0")`) if integrated with a validation framework.  
- **`equals()/hashCode()`** – Implement these if instances will be stored in collections or compared; otherwise, rely on identity semantics.  
- **`toString()`** – Override for better logging/debugging output.  
- **Remove `displaySubTotal`** – Prefer formatting on the client side (or use a dedicated DTO that holds both raw and formatted values). Keeping both can lead to inconsistency.  

### Design Choices  
- **Inheritance vs Composition** – The class currently inherits from `ReadableMinimalProduct`. If `ReadableShoppingCartItem` is only used as a DTO for the UI, composition (`private final ReadableMinimalProduct product;`) is often cleaner and avoids breaking encapsulation.  
- **Money Handling** – Using `java.time` or JSR‑354 `Money` could improve correctness, especially for multi‑currency support.  

### Edge Cases  
- **Null `variant`/`variantValue`** – No null‑check logic; downstream code must handle potential `NullPointerException`.  
- **Empty `cartItemattributes`** – An empty list is fine, but callers should be aware it can be `null` if not explicitly set.  

### Future Enhancements  
- **Builder Pattern** – Add a fluent builder to simplify object creation and enforce immutability.  
- **DTO Validation** – Integrate with Bean Validation (JSR‑380) to automatically enforce constraints on serialization/deserialization.  
- **Localized Formatting** – Move `displaySubTotal` logic to a view layer or a dedicated formatter service that considers locale and currency.  

In summary, the class is straightforward and serves its purpose as a data transfer object. Applying the above improvements would increase robustness, maintainability, and clarity of intent.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.shoppingcart;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import com.salesmanager.shop.model.catalog.product.ReadableMinimalProduct;
import com.salesmanager.shop.model.catalog.product.variation.ReadableProductVariation;

/**
 * compatible with v1 version
 * @author c.samson
 *
 */
public class ReadableShoppingCartItem extends ReadableMinimalProduct implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private BigDecimal subTotal;
	private String displaySubTotal;
	private List<ReadableShoppingCartAttribute> cartItemattributes = new ArrayList<ReadableShoppingCartAttribute>();
	
	private ReadableProductVariation variant = null;
	private ReadableProductVariation variantValue = null;

	

	public BigDecimal getSubTotal() {
		return subTotal;
	}
	public void setSubTotal(BigDecimal subTotal) {
		this.subTotal = subTotal;
	}
	public String getDisplaySubTotal() {
		return displaySubTotal;
	}
	public void setDisplaySubTotal(String displaySubTotal) {
		this.displaySubTotal = displaySubTotal;
	}
	public List<ReadableShoppingCartAttribute> getCartItemattributes() {
		return cartItemattributes;
	}
	public void setCartItemattributes(List<ReadableShoppingCartAttribute> cartItemattributes) {
		this.cartItemattributes = cartItemattributes;
	}
	public ReadableProductVariation getVariant() {
		return variant;
	}
	public void setVariant(ReadableProductVariation variant) {
		this.variant = variant;
	}
	public ReadableProductVariation getVariantValue() {
		return variantValue;
	}
	public void setVariantValue(ReadableProductVariation variantValue) {
		this.variantValue = variantValue;
	}

	
	

}



```
