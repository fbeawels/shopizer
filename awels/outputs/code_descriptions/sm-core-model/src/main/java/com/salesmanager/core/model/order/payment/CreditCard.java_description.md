# CreditCard.java

## Review

## 1. Summary  

The **`CreditCard`** class is a lightweight JPA **embeddable** entity that captures the essential attributes of a credit‑card payment method. It is intended to be nested inside a larger `Payment` or `Order` entity.  

* **Purpose** – Store the card type, cardholder name, number, expiry date and CVV for processing an order.  
* **Key components**  
  * JPA annotations (`@Embeddable`, `@Column`, `@Enumerated`) that map each field to a column in the owning table.  
  * A reference to the `CreditCardType` enum, which represents known card brands.  
  * Standard JavaBean getters and setters.  
* **Notable patterns / libraries** –  
  * **JPA Embeddable**: promotes reuse of the credit‑card mapping across multiple entities.  
  * **Enum mapping**: `@Enumerated(EnumType.STRING)` stores the enum constant name, improving readability and future‑proofing against ordinal changes.  

## 2. Detailed Description  

### Core fields  

| Field | JPA column | Type | Notes |
|-------|------------|------|-------|
| `cardType` | `CARD_TYPE` | `CreditCardType` enum | Stored as string; ensures the DB contains the enum name. |
| `ccOwner` | `CC_OWNER` | `String` | Cardholder name. |
| `ccNumber` | `CC_NUMBER` | `String` | Full card number (should be encrypted). |
| `ccExpires` | `CC_EXPIRES` | `String` | Expiry month/year (format not enforced). |
| `ccCvv` | `CC_CVV` | `String` | 3‑ or 4‑digit security code (sensitive). |

### Execution flow  

1. **Initialization** – When an entity containing `CreditCard` is persisted, JPA creates the column mapping automatically.  
2. **Runtime behavior** – The class is a pure data holder; there is no business logic. It relies on the owning entity for validation and processing.  
3. **Cleanup** – None; it is managed by the JPA provider.  

### Assumptions & constraints  

* The owning entity is responsible for **validating** the card data (format, Luhn check, expiry).  
* Sensitive fields (`ccNumber`, `ccCvv`) are stored in plain text unless encryption is handled elsewhere.  
* No custom `equals`/`hashCode` or `toString` implementations – default identity semantics are used.  

### Architecture & design choices  

* **Reusability** – `@Embeddable` allows the same mapping to be reused across multiple payment or order entities.  
* **Simplicity** – By keeping only getters/setters, the class remains a true value object, delegating business logic elsewhere.  
* **Extensibility** – New fields (e.g., billing address) can be added without affecting the owning entity’s structure.  

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side effects |
|--------|---------|------------|--------|--------------|
| `getCcOwner()` | Retrieve cardholder name. | – | `String` | None |
| `setCcOwner(String ccOwner)` | Set cardholder name. | `String ccOwner` | – | Updates field |
| `getCcNumber()` | Retrieve full card number. | – | `String` | None |
| `setCcNumber(String ccNumber)` | Set full card number. | `String ccNumber` | – | Updates field |
| `getCcExpires()` | Retrieve expiry date string. | – | `String` | None |
| `setCcExpires(String ccExpires)` | Set expiry date string. | `String ccExpires` | – | Updates field |
| `getCcCvv()` | Retrieve CVV code. | – | `String` | None |
| `setCcCvv(String ccCvv)` | Set CVV code. | `String ccCvv` | – | Updates field |
| `getCardType()` | Retrieve card brand enum. | – | `CreditCardType` | None |
| `setCardType(CreditCardType cardType)` | Set card brand enum. | `CreditCardType cardType` | – | Updates field |

These are plain JavaBean accessors; no additional logic is present.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.*` | JPA standard API | Provides `@Embeddable`, `@Column`, `@Enumerated`, etc. |
| `com.salesmanager.core.model.payments.CreditCardType` | Project-specific enum | Defines supported card brands (e.g., VISA, MC). |

All dependencies are either part of the Java EE / Jakarta Persistence specification or belong to the same codebase.

## 5. Additional Notes  

### Security & Compliance  
* **Sensitive data** – Storing `ccNumber` and `ccCvv` as plain strings violates PCI‑DSS guidelines. It is recommended to either:  
  * Encrypt these fields at rest (e.g., AES, JPA AttributeConverter).  
  * Store only a masked version or a token, delegating the full number to a payment gateway.  
* **CVV storage** – PCI‑DSS forbids storing CVV after authorization. If this class is used for long‑term persistence, remove `ccCvv` or mark it `@Transient`.  

### Validation  
* No field validation is performed. Consider adding:  
  * `@NotNull`, `@Size`, `@Pattern` annotations (Hibernate Validator).  
  * Custom logic to validate Luhn checksum for `ccNumber`.  
  * Expiry date format checking (MM/YY or MM/YYYY).  

### Equality & Debugging  
* Implement `equals`, `hashCode`, and `toString` (obscuring sensitive fields) to aid in debugging and collection handling.  

### Immutability  
* For security and thread‑safety, an immutable value object pattern (final fields, constructor injection, no setters) could be adopted.  

### Future Enhancements  
* **Address association** – Link a billing address to the credit card.  
* **Tokenization** – Replace raw card data with a payment token from a gateway.  
* **Internationalization** – Store cardholder name in Unicode, handle locale-specific formatting for expiry.  

Overall, the class fulfills its role as a simple, JPA‑friendly representation of a credit card, but should be augmented with security, validation, and possibly immutability considerations for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.order.payment;

import javax.persistence.Column;
import javax.persistence.Embeddable;
import javax.persistence.EnumType;
import javax.persistence.Enumerated;

import com.salesmanager.core.model.payments.CreditCardType;

@Embeddable
public class CreditCard {
	
	@Column (name ="CARD_TYPE")
	@Enumerated(value = EnumType.STRING)
	private CreditCardType cardType;
	
	@Column (name ="CC_OWNER")
	private String ccOwner;
	
	@Column (name ="CC_NUMBER")
	private String ccNumber;
	
	@Column (name ="CC_EXPIRES")
	private String ccExpires;
	
	@Column (name ="CC_CVV")
	private String ccCvv;

	public String getCcOwner() {
		return ccOwner;
	}

	public void setCcOwner(String ccOwner) {
		this.ccOwner = ccOwner;
	}

	public String getCcNumber() {
		return ccNumber;
	}

	public void setCcNumber(String ccNumber) {
		this.ccNumber = ccNumber;
	}

	public String getCcExpires() {
		return ccExpires;
	}

	public void setCcExpires(String ccExpires) {
		this.ccExpires = ccExpires;
	}

	public String getCcCvv() {
		return ccCvv;
	}

	public void setCcCvv(String ccCvv) {
		this.ccCvv = ccCvv;
	}

	public void setCardType(CreditCardType cardType) {
		this.cardType = cardType;
	}

	public CreditCardType getCardType() {
		return cardType;
	}

}



```
