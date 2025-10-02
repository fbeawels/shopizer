# LightPersistableProduct.java

## Review

## 1. Summary  
**Purpose** – The `LightPersistableProduct` class represents a lightweight, serialisable snapshot of a product’s key state (price, availability, shippability, and quantity). It is intended to be used wherever a reduced representation of a full `Product` entity is sufficient, such as DTOs for API responses, caching, or messaging.

**Key Components**  
- **Fields** – `price`, `available`, `productShipeable`, `quantity`.  
- **Getters/Setters** – Standard JavaBean accessors for all fields.  
- **Serialization** – Implements `Serializable` with a fixed `serialVersionUID`.

**Notable Design Choices**  
- Uses a plain Java class rather than Lombok or generated DTOs.  
- The class is intentionally lightweight (no persistence annotations, no complex logic).  
- The only external contract is `Serializable`, which is a standard Java interface.

---

## 2. Detailed Description  
1. **Initialization** – No constructor is defined; the default no‑arg constructor is used.  
2. **Runtime Behaviour** – The class acts purely as a data holder. Each setter mutates the internal state; getters expose the current state. No validation or business logic is performed.  
3. **Cleanup** – None required; the class has no external resources.  

**Assumptions & Constraints**  
- The price is stored as a `String`. This presumes the caller guarantees the format (e.g., decimal with a currency symbol). In many contexts, a numeric type such as `BigDecimal` would be preferable.  
- `productShipeable` is a misspelling of “shippable”. The field and accessor are consistently named but will cause confusion for future developers.  
- The setter for `productShipeable` accepts a `Boolean` object rather than the primitive `boolean` used elsewhere, which is inconsistent and may lead to unnecessary boxing/unboxing.

**Architecture & Design**  
The class fits into a typical layered architecture where a heavy entity (`Product`) is mapped to a lightweight DTO (`LightPersistableProduct`) for transfer across layers or boundaries. However, the absence of any mapping logic or validation means this class relies entirely on external code to populate it correctly.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getPrice` | `String getPrice()` | Returns the current price string. | None | `String` price | None |
| `setPrice` | `void setPrice(String price)` | Stores the supplied price. | `String price` | None | Mutates `price` field |
| `getQuantity` | `int getQuantity()` | Returns the stored quantity. | None | `int` quantity | None |
| `setQuantity` | `void setQuantity(int quantity)` | Stores the supplied quantity. | `int quantity` | None | Mutates `quantity` field |
| `isAvailable` | `boolean isAvailable()` | Indicates if the product is currently available. | None | `boolean` | None |
| `setAvailable` | `void setAvailable(boolean available)` | Sets the availability flag. | `boolean available` | None | Mutates `available` field |
| `isProductShipeable` | `boolean isProductShipeable()` | Indicates if the product can be shipped. | None | `boolean` | None |
| `setProductShipeable` | `void setProductShipeable(Boolean productShipeable)` | Sets the shippability flag. | `Boolean productShipeable` | None | Mutates `productShipeable` field |

**Reusable/Utility Methods** – None. The class is purely a POJO.

---

## 4. Dependencies  
| Dependency | Type | Remarks |
|------------|------|---------|
| `java.io.Serializable` | Standard Java | Required for Java object serialization. |
| `java.io.Serializable` (serialVersionUID) | Standard | Declares a fixed UID for backward compatibility. |

No external libraries or frameworks are used.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Price Representation** – Using `String` for price opens the door to format errors (e.g., “$10.00”, “10,00”, “USD 10”). A numeric type with explicit currency handling would be safer.  
2. **Typo in Field Name** – `productShipeable` is likely a typo. This will propagate to all generated code (e.g., JSON property names) and can cause confusion or bugs.  
3. **Inconsistent Parameter Types** – The `setProductShipeable` method accepts a wrapper `Boolean` instead of a primitive `boolean`. This inconsistency can lead to `NullPointerException` if a null is passed and may surprise callers.  
4. **Lack of Validation** – No checks are performed (e.g., non‑negative quantity, non‑null price). External code must enforce these invariants.  
5. **No Equality / Hashing** – The class lacks `equals()`, `hashCode()`, and `toString()` overrides. If instances are stored in collections or logged, default `Object` implementations may be insufficient.

### Recommendations  
- **Rename** `productShipeable` to `productShippable`. Update getters/setters accordingly.  
- **Change** the price field to `BigDecimal` (or a custom money type) and optionally include a separate `Currency` field.  
- **Standardise** the setter to use `boolean` rather than `Boolean`.  
- **Add** basic validation in setters (e.g., reject negative quantity).  
- **Implement** `equals()`, `hashCode()`, and `toString()` for better debugging and collection use.  
- **Consider** using Lombok or a record (Java 17+) if the project permits, to reduce boilerplate.  

### Future Enhancements  
- **Mapping Layer** – Create a mapper that converts between `Product` entities and `LightPersistableProduct` DTOs.  
- **Immutability** – Make the DTO immutable by providing a constructor that sets all fields and removing setters, improving thread safety.  
- **Validation Annotations** – If using a framework like Hibernate Validator, annotate fields with constraints (`@NotNull`, `@Positive`).  
- **Serialization Format** – If this object is exposed via REST, use Jackson annotations to control JSON field names and include format hints.  

---

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.catalog.product;

import java.io.Serializable;

/**
 * Lightweight version of Persistable product
 * @author carlsamson
 *
 */
public class LightPersistableProduct implements Serializable {

  /**
   *
   */
  private static final long serialVersionUID = 1L;
  private String price;
  private boolean available;
  private boolean productShipeable;
  private int quantity;
  public String getPrice() {
    return price;
  }
  public void setPrice(String price) {
    this.price = price;
  }

  public int getQuantity() {
    return quantity;
  }
  public void setQuantity(int quantity) {
    this.quantity = quantity;
  }
  public boolean isAvailable() {
    return available;
  }
  public void setAvailable(boolean available) {
    this.available = available;
  }
  public boolean isProductShipeable() { return productShipeable; }
  public void setProductShipeable(Boolean productShipeable) { this.productShipeable = productShipeable; }

}



```
