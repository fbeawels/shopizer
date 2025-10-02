# PaypalPayment.java

## Review

## 1. Summary  
`PaypalPayment` is a **plain‑old Java object (POJO)** that represents a PayPal‑specific payment.  
* It extends a generic `Payment` base class and forces the payment type to `PAYPAL` during construction.  
* Two PayPal‑specific properties are stored:  
  * `payerId` – the identifier of the user on PayPal’s side.  
  * `paymentToken` – the token issued by PayPal’s express checkout flow.  
* The class offers standard getters and setters for these properties.  

No external libraries or frameworks are required; the class is purely domain‑model code.  

---

## 2. Detailed Description  

### Core Components
| Class | Role |
|-------|------|
| `PaypalPayment` | Domain model representing a PayPal payment. Extends `Payment`. |
| `Payment` | (Assumed) base class that holds generic payment information (amount, currency, etc.). |
| `PaymentType` | (Assumed) enum used by `Payment` to indicate the payment method. |

### Execution Flow
1. **Instantiation** – When a `PaypalPayment` is created via `new PaypalPayment()`, the constructor automatically sets the payment type to `PAYPAL` by calling `super.setPaymentType(PaymentType.PAYPAL)`.  
2. **Runtime** – The client populates the `payerId` and `paymentToken` fields via their respective setters. These values are later used by business logic (e.g., payment processing services) to interact with PayPal’s API.  
3. **Cleanup** – No explicit cleanup logic is required; the object is garbage‑collected once it falls out of scope.

### Assumptions & Dependencies
* **Assumption**: `Payment` exposes a `setPaymentType(PaymentType)` method that must be called before other properties are set.  
* **Dependency**: Only the local `Payment` hierarchy and the `PaymentType` enum are needed. No external libs are referenced.  

### Architectural Notes
* The class uses **inheritance** to specialize a generic payment.  
* All state is mutable and public via setters, which is typical for Java domain objects but can lead to inconsistent state if not carefully managed.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `PaypalPayment()` | Constructor – sets the payment type to `PAYPAL`. | – | – | Calls `super.setPaymentType`. |
| `setPayerId(String payerId)` | Stores the PayPal payer ID. | `payerId` – string. | `void` | Mutates `this.payerId`. |
| `getPayerId()` | Retrieves the stored payer ID. | – | `String` | – |
| `setPaymentToken(String paymentToken)` | Stores the PayPal payment token. | `paymentToken` – string. | `void` | Mutates `this.paymentToken`. |
| `getPaymentToken()` | Retrieves the stored payment token. | – | `String` | – |

> **Reusable/Utility Methods** – None beyond the simple getters/setters.  
> **Potential Enhancements** – Adding validation (e.g., non‑null checks), a builder pattern, or making the object immutable.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.model.payments.Payment` | Base class | Internal to the project. |
| `com.salesmanager.core.model.payments.PaymentType` | Enum | Internal to the project. |
| None other | – | No external libraries. |

The class is platform‑agnostic and can be used on any Java runtime that hosts the rest of the `com.salesmanager` domain.

---

## 5. Additional Notes  

### Edge Cases & Validation  
* **Null values** – The current setters accept `null`. Depending on business rules, you might want to enforce non‑null or throw `IllegalArgumentException`.  
* **Format validation** – PayPal IDs and tokens have specific formats; simple regex checks could help catch accidental typos before sending requests to PayPal.  

### Design Alternatives  
* **Composition over inheritance** – Instead of extending `Payment`, consider a `Payment` composition field inside `PaypalPayment`. This can reduce tight coupling and improve flexibility if the payment hierarchy grows.  
* **Immutability** – Creating an immutable `PaypalPayment` (via constructor or builder) would make the object thread‑safe and easier to reason about.  

### Future Enhancements  
* **Serialization** – Implement `Serializable` or JSON annotations (e.g., Jackson) if the object needs to be persisted or sent over the wire.  
* **toString / equals / hashCode** – Auto‑generated methods would aid debugging and collection handling.  
* **Integration tests** – Ensure that `PaypalPayment` correctly interacts with PayPal’s API when used within the payment service layer.  

Overall, the class is concise and fulfills its basic purpose, but adding minimal validation and considering immutability would improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.payments;

/**
 * When the user performs a payment using paypal
 * @author Carl Samson
 *
 */
public class PaypalPayment extends Payment {
	
	//express checkout
	private String payerId;
	private String paymentToken;
	
	public PaypalPayment() {
		super.setPaymentType(PaymentType.PAYPAL);
	}
	
	public void setPayerId(String payerId) {
		this.payerId = payerId;
	}
	public String getPayerId() {
		return payerId;
	}
	public void setPaymentToken(String paymentToken) {
		this.paymentToken = paymentToken;
	}
	public String getPaymentToken() {
		return paymentToken;
	}

}



```
