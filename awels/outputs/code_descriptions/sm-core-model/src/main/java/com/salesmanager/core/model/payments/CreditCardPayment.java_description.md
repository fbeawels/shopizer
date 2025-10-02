# CreditCardPayment.java

## Review

## 1. Summary  

The file defines a **`CreditCardPayment`** class – a simple Java bean that extends an abstract `Payment` base class.  
Its purpose is to carry credit‑card related data (number, CVV, expiry, owner, and card type) for the payment processing subsystem of the *SalesManager* application.  

Key points  
* **POJO/Data‑Transfer Object** – no business logic; only getters/setters.  
* **Extends** a `Payment` superclass (not shown), implying it participates in a polymorphic payment hierarchy.  
* No external frameworks are used; it relies solely on the JDK.  

The code is deliberately minimal, but several issues (naming, data types, security, and missing contract methods) could be improved.

---

## 2. Detailed Description  

### Core components  
| Class | Role |
|-------|------|
| `CreditCardPayment` | Holds credit‑card details for a payment transaction. |

The class contains seven private fields, each with a public getter and setter.  
The class inherits any fields/methods defined in the `Payment` superclass, but those are invisible here.

### Execution flow  
1. **Instantiation** – Typically done by a controller or service that builds a payment request.  
2. **Field population** – Caller sets the card details via setters.  
3. **Processing** – A payment gateway/service consumes the object, reading the values.  
4. **Cleanup** – The object is discarded; no explicit cleanup is present.

### Assumptions & constraints  
* The class assumes that all fields will be populated before use.  
* It trusts the caller to provide a valid card number and CVV.  
* No validation or formatting logic is embedded.  
* It stores sensitive data as plain `String`s, which may conflict with PCI‑DSS compliance expectations.  

### Architecture choice  
The design follows a **“plain data holder”** pattern – common in legacy or lightweight J2EE applications.  
However, in modern Java applications, immutable value objects, builder patterns, or Lombok annotations are preferred to reduce boilerplate and improve safety.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Notes |
|--------|-----------|---------|-------|
| `getCreditCardNumber()` | `String` | Retrieve stored card number | Should be masked or encrypted when exposed |
| `setCreditCardNumber(String)` | void | Assign card number | No validation; risk of storing invalid numbers |
| `getCredidCardValidationNumber()` | `String` | Retrieve CVV | **Typo** – field and method name misspelled |
| `setCredidCardValidationNumber(String)` | void | Assign CVV | Same typo issue |
| `getExpirationMonth()` | `String` | Retrieve expiry month | Should be `int` or an enum |
| `setExpirationMonth(String)` | void | Set expiry month | No range check |
| `getExpirationYear()` | `String` | Retrieve expiry year | Should be `int` |
| `setExpirationYear(String)` | void | Set expiry year | No range check |
| `getCardOwner()` | `String` | Retrieve cardholder name | No formatting rules |
| `setCardOwner(String)` | void | Set cardholder name |  |
| `getCreditCard()` | `CreditCardType` | Retrieve card type enum | May be redundant if number can infer type |
| `setCreditCard(CreditCardType)` | void | Set card type |  |

*All methods are simple accessors; no reusable utilities are present.*

---

## 4. Dependencies  

| Library | Version | Usage |
|---------|---------|-------|
| JDK (java.lang.*) | Standard | Base language features |
| `CreditCardType` | Custom enum | Represents card brand (e.g., VISA, MasterCard) |
| `Payment` | Custom abstract class | Provides common payment fields/methods |

No third‑party libraries, frameworks, or APIs are referenced.

---

## 5. Additional Notes  

### Security & Compliance  
* **Plaintext storage** – The class holds the full card number and CVV in memory as `String`.  
  *PCI‑DSS* requires that the CVV never be stored beyond the authorization request and that card numbers be masked or encrypted when persisted.  
* **No masking** – Getters expose the full values; callers might inadvertently log them.  
* **Sensitive data exposure** – Consider overriding `toString()` to omit or mask sensitive fields.

### Validation & Data Integrity  
* **Typo** – `credidCardValidationNumber` is misspelled; this will propagate to any code that references it.  
* **Field types** – Expiry month/year should be numeric (`int`) or use `java.time.YearMonth`.  
* **Range checks** – No validation of month (1‑12), year (current year‑future), or CVV length.  
* **Card number validation** – No Luhn check or format enforcement.  

### Design Improvements  
1. **Make immutable** – Use a constructor (or builder) that accepts all fields, remove setters.  
2. **Encapsulate sensitive data** – Store masked values or use a dedicated `CardInfo` value object that hides the raw data.  
3. **Lombok or record** – For Java 16+, consider a `record` or use Lombok to reduce boilerplate.  
4. **Documentation** – Add Javadoc comments explaining field constraints and security expectations.  
5. **Unit tests** – Validate that validation logic (once added) correctly rejects bad data.  

### Future Enhancements  
* **Add a `validate()` method** that checks card number format, expiry date, and CVV length.  
* **Implement `equals()`, `hashCode()`, and `toString()`** (with masking).  
* **Integrate with a payment gateway SDK** – map this DTO to the gateway’s request model.  
* **Add support for tokenization** – store a token instead of raw card details, complying with PCI‑DSS.  

---

### Bottom‑Line  
The class serves its immediate purpose as a data holder, but it contains several shortcomings that could lead to data integrity problems, security violations, and maintenance headaches. A refactor to adopt immutability, proper data types, validation, and secure handling of card details would make the component robust and compliant with industry standards.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.payments;

/**
 * When the user performs a payment using a credit card
 * @author Carl Samson
 *
 */
public class CreditCardPayment extends Payment {
	
	private String creditCardNumber;
	private String credidCardValidationNumber;
	private String expirationMonth;
	private String expirationYear;
	private String cardOwner;
	private CreditCardType creditCard;
	public String getCreditCardNumber() {
		return creditCardNumber;
	}
	public void setCreditCardNumber(String creditCardNumber) {
		this.creditCardNumber = creditCardNumber;
	}
	public String getCredidCardValidationNumber() {
		return credidCardValidationNumber;
	}
	public void setCredidCardValidationNumber(String credidCardValidationNumber) {
		this.credidCardValidationNumber = credidCardValidationNumber;
	}
	public String getExpirationMonth() {
		return expirationMonth;
	}
	public void setExpirationMonth(String expirationMonth) {
		this.expirationMonth = expirationMonth;
	}
	public String getExpirationYear() {
		return expirationYear;
	}
	public void setExpirationYear(String expirationYear) {
		this.expirationYear = expirationYear;
	}
	public String getCardOwner() {
		return cardOwner;
	}
	public void setCardOwner(String cardOwner) {
		this.cardOwner = cardOwner;
	}
	public CreditCardType getCreditCard() {
		return creditCard;
	}
	public void setCreditCard(CreditCardType creditCard) {
		this.creditCard = creditCard;
	}

}



```
