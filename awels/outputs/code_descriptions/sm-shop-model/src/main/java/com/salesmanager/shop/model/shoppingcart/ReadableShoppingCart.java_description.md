# ReadableShoppingCart.java

## Review

## 1. Summary  
**Purpose** – `ReadableShoppingCart` is a Data Transfer Object (DTO) used by the **sales‑manager** e‑commerce front‑end to expose shopping cart information in a *read‑only* form (hence the “Readable” prefix).  
**Key Components**  
| Component | Role |
|-----------|------|
| `code` | Unique cart identifier (e.g. session or database key). |
| `subtotal` / `total` | Numeric monetary values of cart contents before/after taxes, discounts etc. |
| `displaySubTotal` / `displayTotal` | Human‑readable formatted strings for UI. |
| `quantity` | Total number of items in the cart. |
| `order` | Link to the associated order once checkout is initiated. |
| `promoCode` | Coupon code applied to the cart. |
| `variant` | Current selected product variant (size, color, etc.). |
| `products` | List of `ReadableShoppingCartItem` – each holds a product and its quantity. |
| `totals` | Detailed breakdown of order totals (`ReadableOrderTotal`), e.g. tax, shipping, discounts. |
| `customer` | Customer identifier (nullable for guest carts). |

**Design Patterns / Libraries** –  
- Plain Java object (POJO) with getters/setters; follows JavaBeans conventions.  
- Uses Java `BigDecimal` for accurate monetary calculations.  
- `ArrayList` as default list implementation.  
- Extends `ShoppingCartEntity` (presumably an ORM entity or base class for cart persistence).  

## 2. Detailed Description  
`ReadableShoppingCart` is a thin wrapper around the underlying cart persistence layer (`ShoppingCartEntity`). It is **immutable to external mutation** because all mutating methods are only getters/setters that expose internal state; business logic should be handled elsewhere (service layer).  

### Flow of Execution  
1. **Instantiation** – Usually created by a service method that fetches a cart from the database and maps the entity fields into this DTO.  
2. **Population** – The service sets each field via the provided setters (or through a builder/constructor if available).  
3. **Return** – The populated DTO is returned to the controller layer or API consumer.  
4. **Cleanup** – No explicit cleanup; relies on Java GC.  

### Assumptions & Constraints  
- Monetary values are represented in the *store’s base currency*; no currency conversion is performed in this DTO.  
- `displaySubTotal` / `displayTotal` are pre‑formatted strings, so formatting is handled upstream (likely a utility).  
- The list fields are mutable; callers can modify the internal list unless defensive copies are returned (not implemented).  
- No validation logic is present; it expects valid data from the service layer.  

### Architecture & Design Choices  
- **Separation of Concerns** – Persistence (`ShoppingCartEntity`) and presentation (`ReadableShoppingCart`) are distinct.  
- **Read‑only Exposure** – The class name implies that it should not be used for write operations.  
- **Flexibility** – By keeping the class generic (no business logic), it can be reused across REST, GraphQL, or other interfaces.  

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `getCustomer()` / `setCustomer(Long)` | Accessor for the customer ID. | `Long` | `Long` | None |
| `getTotals()` / `setTotals(List<ReadableOrderTotal>)` | Accessor for the list of order totals. | `List<ReadableOrderTotal>` | `List<ReadableOrderTotal>` | None |
| `getProducts()` / `setProducts(List<ReadableShoppingCartItem>)` | Accessor for cart items. | `List<ReadableShoppingCartItem>` | `List<ReadableShoppingCartItem>` | None |
| `getCode()` / `setCode(String)` | Cart identifier. | `String` | `String` | None |
| `getSubtotal()` / `setSubtotal(BigDecimal)` | Cart subtotal. | `BigDecimal` | `BigDecimal` | None |
| `getDisplaySubTotal()` / `setDisplaySubTotal(String)` | Formatted subtotal. | `String` | `String` | None |
| `getTotal()` / `setTotal(BigDecimal)` | Cart total. | `BigDecimal` | `BigDecimal` | None |
| `getDisplayTotal()` / `setDisplayTotal(String)` | Formatted total. | `String` | `String` | None |
| `getQuantity()` / `setQuantity(int)` | Total quantity of items. | `int` | `int` | None |
| `getOrder()` / `setOrder(Long)` | Associated order ID. | `Long` | `Long` | None |
| `getPromoCode()` / `setPromoCode(String)` | Applied coupon. | `String` | `String` | None |
| `getVariant()` / `setVariant(ReadableProductVariant)` | Current product variant. | `ReadableProductVariant` | `ReadableProductVariant` | None |

All methods are straightforward getters/setters. There are no utility or business logic methods; that responsibility lies elsewhere.  

## 4. Dependencies  

| Library | Usage | Standard / Third‑Party |
|---------|-------|------------------------|
| `java.math.BigDecimal` | Monetary values | Standard JDK |
| `java.util.ArrayList` / `List` | Collections | Standard JDK |
| `com.salesmanager.shop.model.catalog.product.product.variant.ReadableProductVariant` | Product variant DTO | Application internal |
| `com.salesmanager.shop.model.order.total.ReadableOrderTotal` | Order total breakdown | Application internal |
| `com.salesmanager.shop.model.shoppingcart.ShoppingCartEntity` | Base persistence entity | Application internal |

No external frameworks (Spring, Hibernate, etc.) are directly referenced in this file, but the surrounding application likely uses them.  

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Mutability of Lists** – `getProducts()` and `getTotals()` expose internal mutable lists. External callers could modify the list inadvertently. Consider returning unmodifiable copies or using defensive copying.  
2. **Null Handling** – The class does not enforce non‑null constraints. If fields are null, the UI may break or display "null". Validation should be performed upstream.  
3. **Currency/Locale** – `displaySubTotal`/`displayTotal` assume a single locale/currency format. If the system supports multi‑currency, a separate DTO for localized formatting might be needed.  
4. **Thread Safety** – The object is not thread‑safe; concurrent updates could cause race conditions if the same instance is shared across threads.  

### Future Enhancements  
- **Builder Pattern** – To create immutable instances and reduce the reliance on setters.  
- **Validation Annotations** – Apply `@NotNull`, `@Positive` etc., if using a validation framework.  
- **Immutability** – Convert to an immutable DTO to guarantee thread safety and easier reasoning.  
- **JSON Serialization** – Add Jackson annotations if the DTO is exposed via REST.  
- **Currency Support** – Add fields for currency code and locale or move formatting logic into a view model.  

Overall, the class is clean, self‑explanatory, and fits its role as a simple data container. The main improvement area is defensive handling of mutable collections and potential immutability for safety.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.shoppingcart;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import com.salesmanager.shop.model.catalog.product.product.variant.ReadableProductVariant;
import com.salesmanager.shop.model.order.total.ReadableOrderTotal;

/**
 * Compatible with v1 + v2
 * @author c.samson
 *
 */
public class ReadableShoppingCart extends ShoppingCartEntity {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	private String code;
	private BigDecimal subtotal;


	private String displaySubTotal;
	private BigDecimal total;
	private String displayTotal;
	private int quantity;
	private Long order;
	private String promoCode;
	
	private ReadableProductVariant variant;
	
	List<ReadableShoppingCartItem> products = new ArrayList<ReadableShoppingCartItem>();
	List<ReadableOrderTotal> totals;
	
	private Long customer;



	public Long getCustomer() {
		return customer;
	}



	public void setCustomer(Long customer) {
		this.customer = customer;
	}



	public List<ReadableOrderTotal> getTotals() {
		return totals;
	}



	public void setTotals(List<ReadableOrderTotal> totals) {
		this.totals = totals;
	}



	public List<ReadableShoppingCartItem> getProducts() {
		return products;
	}



	public void setProducts(List<ReadableShoppingCartItem> products) {
		this.products = products;
	}



	public String getCode() {
		return code;
	}



	public void setCode(String code) {
		this.code = code;
	}
	
	public BigDecimal getSubtotal() {
		return subtotal;
	}



	public void setSubtotal(BigDecimal subtotal) {
		this.subtotal = subtotal;
	}



	public String getDisplaySubTotal() {
		return displaySubTotal;
	}



	public void setDisplaySubTotal(String displaySubTotal) {
		this.displaySubTotal = displaySubTotal;
	}



	public BigDecimal getTotal() {
		return total;
	}



	public void setTotal(BigDecimal total) {
		this.total = total;
	}



	public String getDisplayTotal() {
		return displayTotal;
	}



	public void setDisplayTotal(String displayTotal) {
		this.displayTotal = displayTotal;
	}



	public int getQuantity() {
		return quantity;
	}



	public void setQuantity(int quantity) {
		this.quantity = quantity;
	}



	public Long getOrder() {
		return order;
	}



	public void setOrder(Long order) {
		this.order = order;
	}



	public String getPromoCode() {
		return promoCode;
	}



	public void setPromoCode(String promoCode) {
		this.promoCode = promoCode;
	}



	public ReadableProductVariant getVariant() {
		return variant;
	}



	public void setVariant(ReadableProductVariant variant) {
		this.variant = variant;
	}




}



```
