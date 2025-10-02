# ReadableOrder.java

## Review

## 1. Summary

`ReadableOrder` is a **plain‑old Java object (POJO)** that represents a *read‑only* view of an order in the SalesManager shop domain.  
- It extends `OrderEntity`, inheriting core order attributes (id, status, dates, etc.).  
- The class adds additional context such as the customer (`ReadableCustomer`), list of products (`ReadableOrderProduct`), billing/delivery addresses, and a reference to the merchant store.  
- Monetary totals (`total`, `tax`, `shipping`) are stored as `OrderTotal` objects.  
- The class implements `Serializable` and is marked `@Deprecated`, signalling that a newer, preferred representation already exists.

Design-wise, it follows a simple Java Bean pattern with private fields and public getters/setters. No advanced frameworks or patterns are used.

---

## 2. Detailed Description

### Core Components
| Component | Purpose |
|-----------|---------|
| `ReadableCustomer customer` | Holds customer details for the order. |
| `List<ReadableOrderProduct> products` | Ordered items belonging to the order. |
| `Currency currencyModel` | The currency in which the order totals are expressed. |
| `ReadableBilling billing` | Billing address (extends `Address`). |
| `ReadableDelivery delivery` | Delivery address (currently typed as `Address` in the getter, `ReadableDelivery` in the setter). |
| `ReadableMerchantStore store` | The merchant store that fulfilled the order. |
| `OrderTotal total, tax, shipping` | Monetary breakdown of the order. |

### Flow of Execution
1. **Construction / Instantiation** – An instance is usually created by a service or DAO that reads order data from the database, populating the inherited fields from `OrderEntity` and then setting the read‑only attributes via the provided setters.
2. **Runtime** – The object acts as a data transfer object (DTO) between the persistence layer and the REST/JSON representation of an order. Clients read the values via getters; no mutation logic is performed.
3. **Serialization** – Because it implements `Serializable`, the object can be cached or transmitted as a Java‑serialized stream. (However, serialVersionUID is set to 1L, which is fine for a short‑lived API but could cause deserialization issues if the class changes in the future.)

### Assumptions & Constraints
- The class assumes that the caller supplies non‑null objects where necessary. No validation is performed.
- All fields are mutable; callers can change any attribute after construction.
- The `@Deprecated` annotation signals that the rest of the codebase expects a newer representation, likely to remove coupling with the old POJO.

### Architecture & Design Choices
- **Bean-style DTO**: Simple, introspectable, and works well with many Java frameworks (JAXB, Jackson, Hibernate).  
- **Inheritance**: By extending `OrderEntity`, it avoids code duplication but introduces a tight coupling to the persistence layer.  
- **Deprecated Marker**: Indicates that the API is in transition; likely to be replaced by a more type‑safe, immutable DTO in a newer version.

---

## 3. Functions/Methods

| Method | Purpose | Input | Output | Side‑Effects |
|--------|---------|-------|--------|--------------|
| `setCustomer(ReadableCustomer)` | Assigns the customer for the order. | `ReadableCustomer` | void | mutates `customer` |
| `getCustomer()` | Returns the customer. | – | `ReadableCustomer` | none |
| `getTotal()` | Returns the order total. | – | `OrderTotal` | none |
| `setTotal(OrderTotal)` | Sets the order total. | `OrderTotal` | void | mutates `total` |
| `getTax()` | Returns the tax component. | – | `OrderTotal` | none |
| `setTax(OrderTotal)` | Sets the tax component. | `OrderTotal` | void | mutates `tax` |
| `getShipping()` | Returns shipping cost. | – | `OrderTotal` | none |
| `setShipping(OrderTotal)` | Sets shipping cost. | `OrderTotal` | void | mutates `shipping` |
| `getProducts()` | Returns product list. | – | `List<ReadableOrderProduct>` | none |
| `setProducts(List<ReadableOrderProduct>)` | Sets product list. | `List<ReadableOrderProduct>` | void | mutates `products` |
| `setCurrencyModel(Currency)` | Sets currency. | `Currency` | void | mutates `currencyModel` |
| `getBilling()` | Returns billing address. | – | `ReadableBilling` | none |
| `setBilling(ReadableBilling)` | Sets billing address. | `ReadableBilling` | void | mutates `billing` |
| `getDelivery()` | Returns delivery address. | – | `Address` | *Note:* returns base type `Address` but setter accepts `ReadableDelivery`. |
| `setDelivery(ReadableDelivery)` | Sets delivery address. | `ReadableDelivery` | void | mutates `delivery` |
| `getStore()` | Returns merchant store. | – | `ReadableMerchantStore` | none |
| `setStore(ReadableMerchantStore)` | Sets merchant store. | `ReadableMerchantStore` | void | mutates `store` |

**Utility / Reusable Methods**  
The class only contains simple getters/setters; no reusable utilities are defined.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.model.reference.currency.Currency` | Third‑party / Domain | Represents currency metadata. |
| `com.salesmanager.shop.model.customer.*` | Domain | `ReadableCustomer`, `ReadableBilling`, `ReadableDelivery`, `Address`. |
| `com.salesmanager.shop.model.order.OrderEntity` | Domain | Base entity with core order fields. |
| `com.salesmanager.shop.model.order.ReadableOrderProduct` | Domain | DTO for individual order items. |
| `com.salesmanager.shop.model.order.total.OrderTotal` | Domain | DTO encapsulating monetary values. |
| `com.salesmanager.shop.model.store.ReadableMerchantStore` | Domain | Store information. |
| `java.io.Serializable` | Standard | Enables Java serialization. |
| `java.util.List` | Standard | List of order products. |

No external frameworks (e.g., Spring, Hibernate) are referenced directly, but the domain objects may be mapped by ORM or exposed via REST.

---

## 5. Additional Notes

### Strengths
- **Clear separation of concerns**: Order core attributes are inherited; read‑only attributes are added explicitly.
- **Simplicity**: Easy to construct, test, and serialize.
- **Deprecated flag**: Signals to developers that this class should be replaced.

### Weaknesses & Edge Cases
1. **`@Deprecated` Without Replacement** – No guidance on the new API is provided here; developers may struggle to migrate.
2. **Getter/Setter Inconsistency** – `getDelivery()` returns `Address`, but `setDelivery()` accepts `ReadableDelivery`. This mismatch can lead to confusion or runtime type issues.
3. **No Validation** – The setters accept any value; nulls or invalid objects can silently corrupt the DTO.
4. **Thread‑Safety** – The mutable nature means that concurrent access may lead to inconsistent state.
5. **Serialization Fragility** – Changing the class (adding/removing fields) will break compatibility unless `serialVersionUID` is updated carefully.

### Suggested Improvements
- **Use Immutability**: Replace setters with a constructor or builder pattern, making the DTO immutable.
- **Consistent Types**: Align `getDelivery()` to return `ReadableDelivery` or change the setter to accept `Address`.
- **Remove Inheritance**: Prefer composition over inheritance to avoid tight coupling to `OrderEntity`. Introduce a dedicated `OrderSummary` or `OrderDTO`.
- **Explicit Replacement**: Add documentation on the preferred new class (e.g., `OrderReadOnly` or `OrderResponseDto`) and provide migration guidance.
- **Validation & Defensive Copying**: Validate inputs and clone collections to protect internal state.
- **Unit Tests**: Add tests covering all getters/setters and edge cases (null handling, immutability if applied).

### Future Enhancements
- **Builder Pattern** – For complex DTOs, a fluent builder reduces boilerplate.
- **JSON Annotations** – If exposed via REST, annotate fields for Jackson/JSON serialization.
- **Lombok** – Use `@Data`/`@Builder` to reduce boilerplate while maintaining immutability where desired.
- **API Versioning** – Embed API version info in the package or class name to make deprecation explicit.

---

**Bottom line:** `ReadableOrder` is a straightforward DTO that serves a legacy part of the application. Its deprecation flag signals a need for migration. The primary focus for reviewers should be to verify that all clients have moved to the newer representation, address the delivery type mismatch, and consider refactoring to an immutable, composition‑based design for better safety and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.order.v0;

import com.salesmanager.core.model.reference.currency.Currency;
import com.salesmanager.shop.model.customer.ReadableBilling;
import com.salesmanager.shop.model.customer.ReadableCustomer;
import com.salesmanager.shop.model.customer.ReadableDelivery;
import com.salesmanager.shop.model.customer.address.Address;
import com.salesmanager.shop.model.order.OrderEntity;
import com.salesmanager.shop.model.order.ReadableOrderProduct;
import com.salesmanager.shop.model.order.total.OrderTotal;
import com.salesmanager.shop.model.store.ReadableMerchantStore;

import java.io.Serializable;
import java.util.List;

@Deprecated
public class ReadableOrder extends OrderEntity implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private ReadableCustomer customer;
	private List<ReadableOrderProduct> products;
	private Currency currencyModel;
	
	private ReadableBilling billing;
	private ReadableDelivery delivery;
	private ReadableMerchantStore store;
	
	
	
	public void setCustomer(ReadableCustomer customer) {
		this.customer = customer;
	}
	public ReadableCustomer getCustomer() {
		return customer;
	}
	public OrderTotal getTotal() {
		return total;
	}
	public void setTotal(OrderTotal total) {
		this.total = total;
	}
	public OrderTotal getTax() {
		return tax;
	}
	public void setTax(OrderTotal tax) {
		this.tax = tax;
	}
	public OrderTotal getShipping() {
		return shipping;
	}
	public void setShipping(OrderTotal shipping) {
		this.shipping = shipping;
	}

	public List<ReadableOrderProduct> getProducts() {
		return products;
	}
	public void setProducts(List<ReadableOrderProduct> products) {
		this.products = products;
	}

	//public Currency getCurrencyModel() {
	//	return currencyModel;
	//}
	public void setCurrencyModel(Currency currencyModel) {
		this.currencyModel = currencyModel;
	}

	public ReadableBilling getBilling() {
		return billing;
	}
	public void setBilling(ReadableBilling billing) {
		this.billing = billing;
	}

	public Address getDelivery() {
		return delivery;
	}
	public void setDelivery(ReadableDelivery delivery) {
		this.delivery = delivery;
	}

	public ReadableMerchantStore getStore() {
		return store;
	}
	public void setStore(ReadableMerchantStore store) {
		this.store = store;
	}

	private OrderTotal total;
	private OrderTotal tax;
	private OrderTotal shipping;

}



```
