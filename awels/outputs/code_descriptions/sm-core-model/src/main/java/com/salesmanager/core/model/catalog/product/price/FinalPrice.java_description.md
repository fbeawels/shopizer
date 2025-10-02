# FinalPrice.java

## Review

## 1. Summary

The `FinalPrice` class is a **plain‑old Java object (POJO)** that serves as a transient data transfer object (DTO) for rendering price information in the catalog.  
It aggregates several monetary values, flags and helper strings that represent the original, discounted and final prices of a product, along with related metadata (discount percentage, end date, default flag, etc.).  

Key components:
- **`BigDecimal` fields** for precise monetary values (`originalPrice`, `discountedPrice`, `finalPrice`).
- **Boolean flags** (`discounted`, `defaultPrice`) and an **int** (`discountPercent`) to describe discount status.
- **`ProductPrice` reference** linking back to the persistent entity that owns the pricing data.
- **`List<FinalPrice> additionalPrices`** to support price variations (e.g., different currencies or customer segments).

Design-wise, the class is a classic **DTO**—no business logic, only state and accessor methods. No external frameworks or libraries are used beyond the Java SE API.

---

## 2. Detailed Description

### Core responsibilities
1. **State holder** – keep all price‑related values that need to be displayed to the user.
2. **Presentation helper** – expose string representations (`stringPrice`, `stringDiscountedPrice`) for UI layers that may need pre‑formatted values.
3. **Relationship linkage** – reference the underlying `ProductPrice` entity and optionally a list of related `FinalPrice` objects for multi‑variant scenarios.

### Execution flow
- **Initialization** – The class relies on the default no‑arg constructor. The calling code must explicitly populate fields via setters or a builder (not present).
- **Runtime** – The object is typically constructed during price calculation or retrieval from the service layer, filled with data, and passed to the view layer.
- **Cleanup** – No special cleanup is required; it is a simple data holder.

### Assumptions & constraints
- **Null safety** – All numeric fields are nullable; callers must guard against `NullPointerException`.
- **Immutability** – The class is mutable; concurrent access requires external synchronization if used in multi‑threaded contexts.
- **No validation** – Discount percentage is unconstrained; negative or >100 values are silently accepted.
- **Serialization** – Implements `Serializable`; `serialVersionUID` is hardcoded, but the class contains mutable fields, which may lead to versioning issues if the field set changes.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `getAdditionalPrices()` | Retrieve the list of related price objects. | – | `List<FinalPrice>` | – |
| `setAdditionalPrices(List<FinalPrice>)` | Set the list of related price objects. | `List<FinalPrice>` | – | Mutates `additionalPrices`. |
| `getOriginalPrice()` | Get the original price. | – | `BigDecimal` | – |
| `setOriginalPrice(BigDecimal)` | Set the original price. | `BigDecimal` | – | Mutates `originalPrice`. |
| `getDiscountPercent()` | Get discount percentage. | – | `int` | – |
| `setDiscountPercent(int)` | Set discount percentage. | `int` | – | Mutates `discountPercent`. |
| `getDiscountEndDate()` | Get discount expiry date. | – | `Date` | – |
| `setDiscountEndDate(Date)` | Set discount expiry date. | `Date` | – | Mutates `discountEndDate`. |
| `isDiscounted()` | Check if a discount is applied. | – | `boolean` | – |
| `setDiscounted(boolean)` | Mark the price as discounted. | `boolean` | – | Mutates `discounted`. |
| `setDiscountedPrice(BigDecimal)` | Set the discounted price. | `BigDecimal` | – | Mutates `discountedPrice`. |
| `getDiscountedPrice()` | Get the discounted price. | – | `BigDecimal` | – |
| `setFinalPrice(BigDecimal)` | Set the final price after discount. | `BigDecimal` | – | Mutates `finalPrice`. |
| `getFinalPrice()` | Get the final price. | – | `BigDecimal` | – |
| `setDefaultPrice(boolean)` | Mark the price as the default. | `boolean` | – | Mutates `defaultPrice`. |
| `isDefaultPrice()` | Check if this is the default price. | – | `boolean` | – |
| `setProductPrice(ProductPrice)` | Associate the underlying persistent entity. | `ProductPrice` | – | Mutates `productPrice`. |
| `getProductPrice()` | Retrieve the associated `ProductPrice`. | – | `ProductPrice` | – |
| `getStringPrice()` | Get pre‑formatted original price string. | – | `String` | – |
| `setStringPrice(String)` | Set pre‑formatted original price string. | `String` | – | Mutates `stringPrice`. |
| `getStringDiscountedPrice()` | Get pre‑formatted discounted price string. | – | `String` | – |
| `setStringDiscountedPrice(String)` | Set pre‑formatted discounted price string. | `String` | – | Mutates `stringDiscountedPrice`. |

*Note:* All methods are simple getters/setters; there are no business‑logic helpers or validation utilities.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `java.io.Serializable` | Standard Java API | Enables object serialization (e.g., for HTTP session storage). |
| `java.math.BigDecimal` | Standard Java API | Precise decimal arithmetic for monetary values. |
| `java.util.Date` | Standard Java API | Holds the discount end date; could be replaced by `java.time` classes. |
| `java.util.List` | Standard Java API | Holds additional `FinalPrice` instances. |
| `ProductPrice` | Project specific | Not defined here; assumed to be an entity in the same module. |
| No external frameworks (e.g., Spring, Lombok) | – | Manual boilerplate. |

---

## 5. Additional Notes

### Strengths
- **Simplicity** – Easy to understand and use as a DTO.
- **Serialization support** – Helpful for caching or session persistence.
- **Explicit fields** – Clear separation of original, discounted, and final prices.

### Weaknesses / Risks
1. **Null‑ability** – All numeric fields can be `null`. Callers must check for `null` before operations, which can lead to `NullPointerException`s if overlooked.
2. **No immutability** – The mutable nature may cause issues in concurrent or functional contexts.
3. **Lack of validation** – Discount percentage or date logic is not enforced; incorrect values may slip through.
4. **`java.util.Date`** – Consider migrating to `java.time.Instant`/`LocalDateTime` for better API ergonomics and immutability.
5. **Redundant fields** – `stringPrice` and `stringDiscountedPrice` duplicate formatting logic; ideally formatting should happen in the view layer, not stored in the DTO.
6. **Recursive list** – `additionalPrices` contains the same type; if not careful, this can produce deep recursion during serialization or logging.
7. **Missing `equals`/`hashCode`/`toString`** – Useful for debugging and collection operations but currently absent.
8. **No constructor or builder** – The only way to create a fully‑initialized object is to call a chain of setters, which can be error‑prone.

### Edge Cases
- Discount end date in the past but `discounted` flag still `true`.
- Discount percent > 100 or negative.
- `finalPrice` not equal to `originalPrice` or `discountedPrice` when expected.
- `additionalPrices` containing `null` entries.

### Potential Enhancements
1. **Introduce a constructor or builder** to enforce required fields (`originalPrice`, `finalPrice`) and optional ones.
2. **Add validation logic** in setters or a dedicated `validate()` method to enforce consistency (e.g., `finalPrice <= originalPrice` if discounted).
3. **Switch to `java.time` API** for date handling.
4. **Implement `equals`, `hashCode`, and `toString`** (or use Lombok's `@Data` if Lombok is acceptable).
5. **Make the class immutable**: use `final` fields and provide only getters; initialize via constructor.
6. **Remove string representations** and let the presentation layer format `BigDecimal` values.
7. **Add Javadoc** to explain each field’s purpose and any business rules.

---

### Final Verdict

`FinalPrice` is a straightforward DTO that fulfils its immediate purpose of carrying price data from the service layer to the view layer.  
For production‑grade robustness, consider adding validation, immutability, and better date handling.  The class would also benefit from a builder or constructor for clearer object creation and from standard Java overrides (`equals`, `hashCode`, `toString`).  Once these enhancements are in place, the DTO will be safer, easier to use, and less error‑prone in a larger application context.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.catalog.product.price;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.Date;
import java.util.List;

/**
 * Transient entity used to display
 * different price information in the catalogue
 * @author Carl Samson
 *
 */
public class FinalPrice implements Serializable {
	
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private BigDecimal discountedPrice = null;//final price if a discount is applied
	private BigDecimal originalPrice = null;//original price
	private BigDecimal finalPrice = null;//final price discount or not
	private boolean discounted = false;
	private int discountPercent = 0;
	private String stringPrice;
	private String stringDiscountedPrice;
	
	private Date discountEndDate = null;
	
	private boolean defaultPrice;
	private ProductPrice productPrice;
	List<FinalPrice> additionalPrices;

	public List<FinalPrice> getAdditionalPrices() {
		return additionalPrices;
	}

	public void setAdditionalPrices(List<FinalPrice> additionalPrices) {
		this.additionalPrices = additionalPrices;
	}

	public BigDecimal getOriginalPrice() {
		return originalPrice;
	}

	public void setOriginalPrice(BigDecimal originalPrice) {
		this.originalPrice = originalPrice;
	}



	public int getDiscountPercent() {
		return discountPercent;
	}

	public void setDiscountPercent(int discountPercent) {
		this.discountPercent = discountPercent;
	}

	public Date getDiscountEndDate() {
		return discountEndDate;
	}

	public void setDiscountEndDate(Date discountEndDate) {
		this.discountEndDate = discountEndDate;
	}

	public boolean isDiscounted() {
		return discounted;
	}

	public void setDiscounted(boolean discounted) {
		this.discounted = discounted;
	}

	public void setDiscountedPrice(BigDecimal discountedPrice) {
		this.discountedPrice = discountedPrice;
	}

	public BigDecimal getDiscountedPrice() {
		return discountedPrice;
	}


	public void setFinalPrice(BigDecimal finalPrice) {
		this.finalPrice = finalPrice;
	}

	public BigDecimal getFinalPrice() {
		return finalPrice;
	}

	public void setDefaultPrice(boolean defaultPrice) {
		this.defaultPrice = defaultPrice;
	}

	public boolean isDefaultPrice() {
		return defaultPrice;
	}

	public void setProductPrice(ProductPrice productPrice) {
		this.productPrice = productPrice;
	}

	public ProductPrice getProductPrice() {
		return productPrice;
	}

	public String getStringPrice() {
		return stringPrice;
	}

	public void setStringPrice(String stringPrice) {
		this.stringPrice = stringPrice;
	}

	public String getStringDiscountedPrice() {
		return stringDiscountedPrice;
	}

	public void setStringDiscountedPrice(String stringDiscountedPrice) {
		this.stringDiscountedPrice = stringDiscountedPrice;
	}

}



```
