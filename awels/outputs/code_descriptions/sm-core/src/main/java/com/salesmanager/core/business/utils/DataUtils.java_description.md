# DataUtils.java

## Review

## 1. Summary  
**Purpose**  
`DataUtils` is a small, static helper class that provides common data manipulation utilities used throughout the SalesManager core module. Its responsibilities include:

1. Cleaning postal code strings by removing non‑alphanumeric characters.  
2. Converting weight values between pounds (LB) and kilograms (KG) based on a store’s configured unit.  
3. Converting linear measurements between inches (IN) and centimeters (CM) based on a store’s configured unit.

**Key Components**  
- Three public static methods: `trimPostalCode`, `getWeight`, and `getMeasure`.  
- Heavy reliance on `java.math.BigDecimal` for rounding to two decimal places.  
- Uses `com.salesmanager.core.constants.MeasureUnit` enum to determine units.  
- Relies on the `MerchantStore` entity for the store’s default unit configuration.

**Design Patterns / Frameworks**  
- *Utility Class* pattern: All methods are static and stateless, intended for quick access across the application.  
- No external frameworks beyond the JDK and the internal SalesManager model.

---

## 2. Detailed Description  
### Flow of Execution  

1. **`trimPostalCode`**  
   - Accepts a `String` that may contain dashes, spaces, or other punctuation.  
   - Uses a regex to strip every non‑alphanumeric character.  
   - Returns the cleaned string.

2. **`getWeight`**  
   - Parameters: raw `weight` (in the base unit), a `MerchantStore` object, and a `String` (`base`) indicating the unit the input value is expressed in.  
   - The method first checks whether the `base` is `LB`.  
     - If the store also uses `LB`, the value is simply rounded to 2 decimal places.  
     - If the store uses `KG`, the weight is converted from pounds to kilograms (multiplication by 2.2).  
   - If the `base` is `KG`, the logic is inverted:  
     - Store’s unit matches → direct rounding.  
     - Store’s unit is `LB` → convert kilograms to pounds by division by 2.2.  
   - All conversions use `BigDecimal` for precision and `RoundingMode.HALF_UP`.

3. **`getMeasure`**  
   - Very similar logic to `getWeight`, but handles linear units (IN ↔ CM).  
   - Conversion constants:  
     - IN → CM: multiply by 2.54.  
     - CM → IN: multiply by 0.39 (approximate inverse of 2.54).  
   - Rounds to two decimal places.

### Assumptions & Constraints  
- The `base` parameter is expected to match the enum names (`LB`, `KG`, `IN`, `CM`). No defensive checks are performed.  
- `MerchantStore.getWeightunitcode()` and `getSeizeunitcode()` return `String`s that exactly match the enum names.  
- The conversion constants are hard‑coded, limiting flexibility (e.g., if international standards change).  
- Rounding is performed every time, which may not be desirable for all use‑cases.

### Architecture & Design Choices  
- A *utility* approach keeps the code simple and avoids dependency injection, but it also makes unit testing harder (though still straightforward due to static methods).  
- Using `BigDecimal` guarantees decimal precision for monetary or measurement calculations, though the initial conversion to `double` can re‑introduce floating‑point inaccuracies before rounding.  
- The design mixes business logic (unit conversion) with data formatting (postal code trimming), which could be split into separate, more focused classes.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Parameters | Returns | Side Effects |
|--------|-----------|---------|------------|---------|--------------|
| `trimPostalCode` | `public static String trimPostalCode(String postalCode)` | Removes all non‑alphanumeric characters from a postal code. | `postalCode` – raw postal code string. | Cleaned postal code. | None |
| `getWeight` | `public static double getWeight(double weight, MerchantStore store, String base)` | Converts `weight` between pounds and kilograms according to store’s unit. | `weight` – raw numeric weight.<br>`store` – store configuration object.<br>`base` – unit of the input (`LB` or `KG`). | Converted weight rounded to 2 decimals. | None |
| `getMeasure` | `public static double getMeasure(double measure, MerchantStore store, String base)` | Converts `measure` between inches and centimeters according to store’s unit. | `measure` – raw numeric measurement.<br>`store` – store configuration object.<br>`base` – unit of the input (`IN` or `CM`). | Converted measurement rounded to 2 decimals. | None |

**Reusable / Utility Methods**  
- The conversion logic could be extracted into private helper methods to avoid duplication between `getWeight` and `getMeasure`.  
- A generic method that accepts a conversion factor and the two unit names could make the code more DRY.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | JDK | Standard library for arbitrary‑precision decimal arithmetic. |
| `java.math.RoundingMode` | JDK | Used for rounding strategy. |
| `com.salesmanager.core.constants.MeasureUnit` | Internal | Enum defining unit codes (`LB`, `KG`, `IN`, `CM`). |
| `com.salesmanager.core.model.merchant.MerchantStore` | Internal | Domain model that holds store configuration (weight and seize unit codes). |

No external third‑party libraries or platform‑specific APIs are used.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

| Scenario | Current Behavior | Suggested Improvement |
|----------|------------------|-----------------------|
| `base` is `null` or not one of the expected values | `NullPointerException` or unexpected behavior | Validate `base` and throw a clear `IllegalArgumentException`. |
| Store unit codes are in lowercase or mixed case | No match, leading to incorrect conversion | Use `equalsIgnoreCase` or store unit codes as enum values rather than strings. |
| Conversion constants (2.2, 2.54, 0.39) are approximate | Potential rounding drift for large values | Store constants as `BigDecimal` and compute `1 / 2.54` exactly. |
| Input weight/measure is negative | Conversion proceeds without validation | Add guard clauses if negative values are invalid in business context. |
| `MerchantStore` is `null` | `NullPointerException` | Add a null check and throw an informative exception. |

### Testing Considerations  

- **Unit tests** should cover: normal conversions, same-unit scenarios, edge cases (zero, negative, large numbers), and null/invalid inputs.  
- Because methods are static, tests can call them directly without mocking.  

### Potential Enhancements  

1. **Introduce a Unit Conversion Service**  
   - Abstract the logic into an injectable service that can be swapped or extended (e.g., to support metric prefixes).  

2. **DRY the conversion logic**  
   - Create a private helper: `private static double convert(double value, double factor)` that handles rounding and returns the result.  

3. **Enum‑based unit handling**  
   - Replace string comparisons with the `MeasureUnit` enum directly, eliminating string matching bugs.  

4. **Use `BigDecimal` for entire calculation**  
   - Avoid converting to `double` before performing the conversion; keep everything in `BigDecimal` to preserve precision.  

5. **Document rounding expectations**  
   - Add JavaDoc comments that clarify why rounding is performed and how it aligns with business rules.

---

### Conclusion  

`DataUtils` delivers a small, focused set of utilities that perform common data transformations. The code is straightforward, but it could benefit from tighter input validation, reduced duplication, and a more type‑safe approach to unit handling. With modest refactoring, the class can become more robust, maintainable, and easier to test.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.utils;

import java.math.BigDecimal;
import java.math.RoundingMode;

import com.salesmanager.core.constants.MeasureUnit;
import com.salesmanager.core.model.merchant.MerchantStore;

public class DataUtils {
	
	/**
	 * Removes dashes
	 * @param postalCode
	 * @return
	 */
	public static String trimPostalCode(String postalCode) {
		return postalCode.replaceAll("[^a-zA-Z0-9]", "");
	}
	
	
	/**
	 * Get the measure according to the appropriate measure base. If the measure
	 * configured in store is LB and it needs KG then the appropriate
	 * calculation is done
	 * 
	 * @param weight
	 * @param store
	 * @param base
	 * @return
	 */
	public static double getWeight(double weight, MerchantStore store,
			String base) {

		double weightConstant = 2.2;
		if (base.equals(MeasureUnit.LB.name())) {
			if (store.getWeightunitcode().equals(MeasureUnit.LB.name())) {
				return new BigDecimal(String.valueOf(weight))
				.setScale(2, RoundingMode.HALF_UP).doubleValue();
			} else {// pound = kilogram
				double answer = weight * weightConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, RoundingMode.HALF_UP).doubleValue();
			}
		} else {// need KG
			if (store.getWeightunitcode().equals(MeasureUnit.KG.name())) {
				return new BigDecimal(String.valueOf(weight)).setScale(2,
				RoundingMode.HALF_UP).doubleValue();
			} else {
				double answer = weight / weightConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, RoundingMode.HALF_UP).doubleValue();
			}
		}
	}
	
	/**
	 * Get the measure according to the appropriate measure base. If the measure
	 * configured in store is IN and it needs CM or vise versa then the
	 * appropriate calculation is done
	 * 
	 * @param weight
	 * @param store
	 * @param base
	 * @return
	 */
	public static double getMeasure(double measure, MerchantStore store,
			String base) {

		if (base.equals(MeasureUnit.IN.name())) {
			if (store.getSeizeunitcode().equals(MeasureUnit.IN.name())) {
				return new BigDecimal(String.valueOf(measure)).setScale(2,
				RoundingMode.HALF_UP).doubleValue();
			} else {// centimeter (inch to centimeter)
				double measureConstant = 2.54;

				double answer = measure * measureConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, RoundingMode.HALF_UP).doubleValue();

			}
		} else {// need CM
			if (store.getSeizeunitcode().equals(MeasureUnit.CM.name())) {
				return new BigDecimal(String.valueOf(measure)).setScale(2,
				RoundingMode.HALF_UP).doubleValue();
			} else {// in (centimeter to inch)
				double measureConstant = 0.39;

				double answer = measure * measureConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, RoundingMode.HALF_UP).doubleValue();

			}
		}

	}

}



```
