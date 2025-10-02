# PersistableOrder.java

## Review

## 1. Summary  

The `PersistableOrder` class is a lightweight data‑transfer object (DTO) used during the order processing workflow in the **SalesManager** e‑commerce platform. It extends the base `Order` entity and adds a few fields that are required for persistence and payment handling:

| Field | Purpose |
|-------|---------|
| `PersistablePayment payment` | Holds payment information that will be persisted alongside the order. |
| `Long shippingQuote` | Identifier of the selected shipping quote. |
| `Long shoppingCartId` | Identifier of the shopping cart that generated this order. |
| `Long customerId` | Identifier of the customer placing the order. |

The class is annotated with Jackson’s `@JsonIgnore` on the cart and customer identifiers so that those values are omitted when the object is serialized to JSON (e.g., when the API returns the order payload).

**Key components & patterns**

- **Inheritance**: Extends `Order` to reuse all core order fields (e.g., billing, shipping address, items, totals).
- **DTO pattern**: Provides a mutable object that can be populated by the controller layer, validated, and then handed off to the service layer for persistence.
- **Jackson annotations**: `@JsonIgnore` to control JSON representation.
- **Serializable**: Implements Java serialization (via the parent `Order`) with a generated `serialVersionUID`.

The class is intentionally simple, acting as a container rather than containing business logic.

---

## 2. Detailed Description  

### Core responsibilities

1. **Persistable Order Representation** – The object aggregates all data needed to create or update an order in the database.  
2. **Payment Attachment** – `PersistablePayment` is a separate DTO that carries payment specifics (card details, transaction status, etc.).  
3. **Meta‑data Storage** – `shippingQuote`, `shoppingCartId`, and `customerId` provide context for the order (origin, chosen shipping, user).  
4. **JSON Control** – By ignoring `shoppingCartId` and `customerId`, the API ensures sensitive internal IDs are not exposed to the client, while still keeping them available for internal processing.

### Execution flow

1. **Request handling** – An incoming order request is deserialized into a `PersistableOrder` instance by Spring MVC (or another framework).  
2. **Validation** – The service layer validates all fields (e.g., non‑null payment, correct shipping quote).  
3. **Business logic** – The order is processed: totals are recalculated, inventory is reserved, the payment transaction is executed, etc.  
4. **Persistence** – The service converts the DTO into a JPA entity (`OrderEntity`) and persists it via the repository.  
5. **Response** – The persisted entity is mapped back to a read‑only DTO and returned to the client.

### Design choices & assumptions

- **Mutable DTO** – The class is intentionally mutable to simplify mapping from request payloads.  
- **Separation of concerns** – All domain logic (payment, inventory, pricing) lives in the service layer, not here.  
- **Use of Jackson annotations** – The API relies on Jackson for JSON serialization; `@JsonIgnore` is a quick way to hide internal IDs.  
- **No validation annotations** – The code does not use Bean Validation (`@NotNull`, `@Size`, …). It assumes validation is performed elsewhere.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `getShoppingCartId()` | Retrieve the shopping cart identifier. | – | `Long` | None |
| `setShoppingCartId(Long)` | Set the shopping cart identifier. | `shoppingCartId` | `void` | Updates internal state |
| `getCustomerId()` | Retrieve the customer identifier. | – | `Long` | None |
| `setCustomerId(Long)` | Set the customer identifier. | `customerId` | `void` | Updates internal state |
| `getPayment()` | Retrieve the payment DTO. | – | `PersistablePayment` | None |
| `setPayment(PersistablePayment)` | Set the payment DTO. | `payment` | `void` | Updates internal state |
| `getShippingQuote()` | Retrieve the shipping quote id. | – | `Long` | None |
| `setShippingQuote(Long)` | Set the shipping quote id. | `shippingQuote` | `void` | Updates internal state |

> **Note**: All methods are standard getters/setters. No business logic is implemented within the class.

---

## 4. Dependencies  

| Dependency | Type | Usage |
|------------|------|-------|
| `com.fasterxml.jackson.annotation.JsonIgnore` | Third‑party (Jackson) | Controls JSON serialization of specific fields. |
| `com.salesmanager.shop.model.order.transaction.PersistablePayment` | In‑project | DTO representing payment details. |
| `java.io.Serializable` | Standard | Enables serialization (via `serialVersionUID`). |

There are **no other external libraries** referenced directly in this class. It relies on the surrounding Spring/REST infrastructure for JSON mapping and dependency injection.

---

## 5. Additional Notes  

### Strengths  

- **Clarity** – The purpose of each field is self‑explanatory.  
- **Simplicity** – Minimal code makes maintenance straightforward.  
- **Separation** – Keeps DTOs distinct from entities and business logic.

### Potential Issues & Edge Cases  

1. **Null Handling**  
   - The class does not guard against `null` values. If a caller passes `null` for `payment` or `shippingQuote`, downstream services must handle it, or risk `NullPointerException`.  
2. **Validation**  
   - Bean Validation annotations are missing. Without them, the system depends entirely on manual checks elsewhere.  
3. **Serialization Consistency**  
   - `@JsonIgnore` hides two fields in responses, but the same class is also used for incoming requests. If a client omits `shoppingCartId` or `customerId`, they must be supplied by other means (e.g., session context).  
4. **Equality & Hashing**  
   - No `equals()`, `hashCode()`, or `toString()` overrides are provided. If these objects are ever used in collections or logs, the default implementation may not be helpful.  
5. **Immutability**  
   - Mutable DTOs can lead to accidental state changes. If the application uses concurrent processing, consider making the DTO immutable or using defensive copies.

### Suggested Enhancements  

| Enhancement | Benefit |
|-------------|---------|
| Add Bean Validation (`@NotNull`, `@Min`, etc.) | Early detection of malformed requests. |
| Implement `equals()`, `hashCode()`, `toString()` | Better debugging and collection handling. |
| Introduce a builder pattern | More robust object creation, especially when many optional fields exist. |
| Convert to an immutable value object | Reduce accidental mutation and improve thread‑safety. |
| Centralize JSON visibility using views or mix‑ins | Avoid mixing business‑logic fields with serialization control. |
| Add Javadoc for public methods | Improve developer experience and maintainability. |

--- 

**Bottom line** – `PersistableOrder` is a straightforward, well‑intentioned DTO that cleanly separates API representation from domain logic. The class is functional as‑is for a small code base, but adding validation, immutability, and better object semantics would make it more robust for larger, production‑grade systems.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.order.v1;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.shop.model.order.transaction.PersistablePayment;

/**
 * This object is used when processing an order from the API
 * It will be used for processing the payment and as Order meta data
 * @author c.samson
 *
 */
public class PersistableOrder extends Order {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	private PersistablePayment payment;
	private Long shippingQuote;
	@JsonIgnore
	private Long shoppingCartId;
	@JsonIgnore
	private Long customerId;
	
	
	
	public Long getShoppingCartId() {
		return shoppingCartId;
	}

	public void setShoppingCartId(Long shoppingCartId) {
		this.shoppingCartId = shoppingCartId;
	}

	public Long getCustomerId() {
		return customerId;
	}

	public void setCustomerId(Long customerId) {
		this.customerId = customerId;
	}

	public PersistablePayment getPayment() {
		return payment;
	}

	public void setPayment(PersistablePayment payment) {
		this.payment = payment;
	}

	public Long getShippingQuote() {
		return shippingQuote;
	}

	public void setShippingQuote(Long shippingQuote) {
		this.shippingQuote = shippingQuote;
	}
	


}



```
