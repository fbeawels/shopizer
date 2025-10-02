# Order.java

## Review

## 1. Summary  
**Purpose** – The `Order` entity represents a customer purchase in the SalesManager system.  
It captures all information required to persist an order: identification, status, dates, customer references, payment and shipping details, currency, totals, addresses, and collections of related objects such as products, totals, status history and custom attributes.

**Key components**  

| Component | Role |
|-----------|------|
| `@Entity @Table(name="ORDERS")` | Declares a JPA entity that maps to the `ORDERS` table. |
| `id` | Primary key generated via a table‑based sequence (`SM_SEQUENCER`). |
| `status`, `channel`, `orderType`, `paymentType` | Enum fields stored as strings. |
| `datePurchased`, `orderDateFinished`, `lastModified` | Date/timestamp fields; dates are defensive‑cloned in getters/setters. |
| `currency`, `locale` | Relations to `Currency` and a custom Hibernate `locale` type. |
| `merchant` | The owning `MerchantStore`. |
| `orderProducts`, `orderTotal`, `orderHistory`, `orderAttributes` | One‑to‑many collections with cascade ALL and explicit ordering. |
| `delivery`, `billing`, `creditCard` | Embedded value objects holding shipping, billing and (deprecated) card data. |
| `customerId`, `customerEmailAddress` | Basic customer linkage, intentionally not a JPA relation to allow orphaned customers. |

**Notable patterns / libraries**  
* JPA/Hibernate for persistence.  
* Hibernate’s `@OrderBy` and custom `@Type` for `Locale`.  
* Jackson’s `@JsonIgnore` to hide the `merchant` in JSON.  
* Defensive copying of `java.util.Date` via `CloneUtils`.  
* Extends `SalesManagerEntity<Long, Order>` – a common base for all entities in the project.

---

## 2. Detailed Description  

### Entity lifecycle  
* **Creation** – `Order()` creates a new instance with defaults (e.g., `total`, `currencyValue`, `orderType`).  
* **Persistence** – Hibernate populates the fields according to the annotations. The `id` is generated via a table generator.  
* **Runtime** – Business logic manipulates the entity through the exposed getters/setters. The collections are `LinkedHashSet`s so insertion order is preserved unless overridden by `@OrderBy`.  
* **Serialization** – Jackson will serialize all properties except `merchant` (ignored).  
* **Cleanup** – No explicit cleanup logic; cascading will persist or remove child collections automatically.  

### Inter‑entity relationships  
* `MerchantStore` – Many‑to‑one; default fetch is EAGER which can lead to unnecessary joins.  
* `Currency` – Many‑to‑one; also EAGER.  
* Collections (`OrderProduct`, `OrderTotal`, `OrderStatusHistory`, `OrderAttribute`) – One‑to‑many with cascade ALL. No `orphanRemoval`, so deleted items may remain in the DB unless handled elsewhere.  

### Assumptions / constraints  
* `customerId` is a simple FK; the corresponding `Customer` entity can be removed without breaking the order.  
* `delivery`, `billing`, and `creditCard` are value objects and embedded; they cannot exist independently.  
* Dates are cloned defensively to avoid mutability leaks.  
* Currency and locale values are assumed valid and non‑null.  

### Architectural choices  
* Using a **table generator** (`SM_SEQUENCER`) instead of DB sequences for portability.  
* Storing dates as `java.util.Date` (instead of the newer `java.time` API) for backward compatibility.  
* Embedding address and card data keeps the order entity self‑contained.  
* Enumerations are stored as strings to keep the DB readable.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `getId()/setId(Long)` | PK accessors (overridden from base). | `Long` | `Long` | Sets/gets `id`. |
| `getStatus()/setStatus(OrderStatus)` | Order status accessors. | `OrderStatus` | `OrderStatus` |  |
| `getLastModified()/setLastModified(Date)` | Defensive clone for `lastModified`. | `Date` | `Date` |  |
| `getDatePurchased()/setDatePurchased(Date)` | Defensive clone for `datePurchased`. | `Date` | `Date` |  |
| `getOrderDateFinished()/setOrderDateFinished(Date)` | Defensive clone for `orderDateFinished`. | `Date` | `Date` |  |
| `getCurrencyValue()/setCurrencyValue(BigDecimal)` | Exchange rate accessor. | `BigDecimal` | `BigDecimal` |  |
| `getTotal()/setTotal(BigDecimal)` | Order total accessor. | `BigDecimal` | `BigDecimal` |  |
| `getIpAddress()/setIpAddress(String)` | IP address accessor. | `String` | `String` |  |
| `getPaymentModuleCode()/setPaymentModuleCode(String)` | Payment module accessor. | `String` | `String` |  |
| `getShippingModuleCode()/setShippingModuleCode(String)` | Shipping module accessor. | `String` | `String` |  |
| `getCurrency()/setCurrency(Currency)` | Currency accessor. | `Currency` | `Currency` |  |
| `getMerchant()/setMerchant(MerchantStore)` | Merchant accessor. | `MerchantStore` | `MerchantStore` |  |
| `getOrderProducts()/setOrderProducts(Set<OrderProduct>)` | Product collection. | `Set<OrderProduct>` | `Set<OrderProduct>` |  |
| `getOrderTotal()/setOrderTotal(Set<OrderTotal>)` | Totals collection. | `Set<OrderTotal>` | `Set<OrderTotal>` |  |
| `getOrderHistory()/setOrderHistory(Set<OrderStatusHistory>)` | Status history. | `Set<OrderStatusHistory>` | `Set<OrderStatusHistory>` |  |
| `setDelivery(Delivery)` / `getDelivery()` | Shipping address. | `Delivery` | `Delivery` |  |
| `setBilling(Billing)` / `getBilling()` | Billing address. | `Billing` | `Billing` |  |
| `setCreditCard(CreditCard)` / `getCreditCard()` | Card data (deprecated). | `CreditCard` | `CreditCard` |  |
| `getCustomerId()/setCustomerId(Long)` | Customer FK. | `Long` | `Long` |  |
| `getCustomerEmailAddress()/setCustomerEmailAddress(String)` | Customer email. | `String` | `String` |  |
| `setChannel(OrderChannel)` / `getChannel()` | Order channel. | `OrderChannel` | `OrderChannel` |  |
| `setPaymentType(PaymentType)` / `getPaymentType()` | Payment type. | `PaymentType` | `PaymentType` |  |
| `getOrderType()/setOrderType(OrderType)` | Order type (ORDER/QUOTE etc.). | `OrderType` | `OrderType` |  |
| `getLocale()/setLocale(Locale)` | Locale. | `Locale` | `Locale` |  |
| `getCustomerAgreement()/setCustomerAgreement(Boolean)` | Customer agreement flag. | `Boolean` | `Boolean` |  |
| `getConfirmedAddress()/setConfirmedAddress(Boolean)` | Address confirmation flag. | `Boolean` | `Boolean` |  |
| `getOrderAttributes()/setOrderAttributes(Set<OrderAttribute>)` | Custom attributes. | `Set<OrderAttribute>` | `Set<OrderAttribute>` |  |
| `getShoppingCartCode()/setShoppingCartCode(String)` | Shopping cart code. | `String` | `String` |  |

All setters perform simple assignments; only date setters clone to avoid exposing mutable `Date` objects.  

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.*` | JPA (standard) | Core entity annotations. |
| `org.hibernate.annotations.*` | Hibernate (3rd‑party) | `@OrderBy`, `@Type` for `Locale`. |
| `com.fasterxml.jackson.annotation.JsonIgnore` | Jackson (3rd‑party) | Prevents serialization of the merchant. |
| `com.salesmanager.core.model.*` | Domain (internal) | All related entities (`Billing`, `Delivery`, `OrderProduct`, etc.). |
| `org.hibernate.validator.*` (via `@Valid`) | Validation (3rd‑party) | Validates embedded `Billing`. |
| `com.salesmanager.core.utils.CloneUtils` | Internal utility | Defensive cloning of `Date`. |
| `java.util.*` | Standard | Date, Set, Locale, etc. |
| `java.math.BigDecimal` | Standard | Monetary values. |

No external framework beyond JPA/Hibernate and Jackson.  

---

## 5. Additional Notes & Recommendations  

### Performance & Mapping  
1. **Fetch strategy** – `@ManyToOne` defaults to EAGER; consider `fetch = FetchType.LAZY` for `currency`, `merchant`, and any other associations that are not always required.  
2. **Orphan removal** – Cascading ALL on collections is fine, but if a child is removed from the set it will remain in the DB. Add `orphanRemoval = true` where appropriate.  
3. **Table generator** – While portable, it can become a bottleneck under heavy write load. If the database supports sequences, switch to `GenerationType.SEQUENCE`.  

### Code Quality  
1. **Use of `java.util.Date`** – Replace with `java.time.Instant` / `ZonedDateTime` for immutability and better semantics.  
2. **Equality / hash** – Implement `equals()` and `hashCode()` based on business key (`id`) or use Lombok’s `@EqualsAndHashCode`.  
3. **`@Deprecated` field** – `creditCard` is marked deprecated but still persists; decide whether to remove or migrate to a payment gateway entity.  
4. **Validation** – Add JSR‑380 constraints (e.g., `@Email` for `customerEmailAddress`, `@NotBlank` for mandatory fields).  
5. **Builder pattern** – To reduce boilerplate and improve readability, consider Lombok’s `@Builder` or a custom builder.  

### Security & Data Integrity  
1. **Sensitive data** – `creditCard` fields are stored as embedded but not masked; ensure encryption or tokenization if real card numbers are persisted.  
2. **Audit trail** – `lastModified` is cloned but not automatically updated. Use `@PreUpdate`/`@PrePersist` callbacks or Hibernate’s `@CreationTimestamp` / `@UpdateTimestamp`.  

### Extensibility  
* Adding new order types or channels should be straightforward as they are enums.  
* For multi‑currency pricing, consider storing prices in base currency and converting on the fly.  
* The `orderAttributes` set provides a flexible way to attach custom key/value pairs; document how this is consumed by the application.

### Edge Cases  
* **Null `locale`** – The `@Type(type="locale")` may fail if `locale` is null; handle gracefully.  
* **Large collections** – If an order contains many products, the `LinkedHashSet` could grow large; consider pagination or lazy loading.  
* **Order deletion** – With cascade ALL and no orphan removal, deleting an order may leave orphaned `OrderProduct` records; ensure proper cleanup logic.  

---

### Bottom‑line  
The `Order` entity is a solid, conventional JPA model that covers the majority of e‑commerce order requirements. It is well‑structured, but a few modernizations (lazy loading, immutable dates, proper orphan handling, and clearer serialization rules) would enhance performance, security, and maintainability. With those adjustments, the class would be ready for high‑volume, production‑grade usage.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.order;

import java.math.BigDecimal;
import java.util.Date;
import java.util.LinkedHashSet;
import java.util.Locale;
import java.util.Set;

import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.EnumType;
import javax.persistence.Enumerated;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.JoinColumn;
import javax.persistence.ManyToOne;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.validation.Valid;

import org.hibernate.annotations.OrderBy;
import org.hibernate.annotations.Type;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.core.model.common.Billing;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.attributes.OrderAttribute;
import com.salesmanager.core.model.order.orderproduct.OrderProduct;
import com.salesmanager.core.model.order.orderstatus.OrderStatus;
import com.salesmanager.core.model.order.orderstatus.OrderStatusHistory;
import com.salesmanager.core.model.order.payment.CreditCard;
import com.salesmanager.core.model.payments.PaymentType;
import com.salesmanager.core.model.reference.currency.Currency;
import com.salesmanager.core.utils.CloneUtils;

@Entity
@Table (name="ORDERS")
public class Order extends SalesManagerEntity<Long, Order> {
	
	
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;

	@Id
	@Column (name ="ORDER_ID" , unique=true , nullable=false )
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT",
		pkColumnValue = "ORDER_ID_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;
	
	@Column (name ="ORDER_STATUS")
	@Enumerated(value = EnumType.STRING)
	private OrderStatus status;
	
	@Temporal(TemporalType.TIMESTAMP)
	@Column (name ="LAST_MODIFIED")
	private Date lastModified;
	
	//the customer object can be detached. An order can exist and the customer deleted
	@Column (name ="CUSTOMER_ID")
	private Long customerId;
	
	@Temporal(TemporalType.DATE)
	@Column (name ="DATE_PURCHASED")
	private Date datePurchased;
	
	//used for an order payable on multiple installment
	@Temporal(TemporalType.TIMESTAMP)
	@Column (name ="ORDER_DATE_FINISHED")
	private Date orderDateFinished;
	
	//What was the exchange rate
	@Column (name ="CURRENCY_VALUE")
	private BigDecimal currencyValue = new BigDecimal(1);//default 1-1
	
	@Column (name ="ORDER_TOTAL")
	private BigDecimal total;

	@Column (name ="IP_ADDRESS")
	private String ipAddress;
	
	@Column(name = "CART_CODE", nullable=true)
	private String shoppingCartCode;

	@Column (name ="CHANNEL")
	@Enumerated(value = EnumType.STRING)
	private OrderChannel channel;

	@Column (name ="ORDER_TYPE")
	@Enumerated(value = EnumType.STRING)
	private OrderType orderType = OrderType.ORDER;

	@Column (name ="PAYMENT_TYPE")
	@Enumerated(value = EnumType.STRING)
	private PaymentType paymentType;
	
	@Column (name ="PAYMENT_MODULE_CODE")
	private String paymentModuleCode;
	
	@Column (name ="SHIPPING_MODULE_CODE")
	private String shippingModuleCode;
	
	@Column(name = "CUSTOMER_AGREED")
	private Boolean customerAgreement = false;
	
	@Column(name = "CONFIRMED_ADDRESS")
	private Boolean confirmedAddress = false;

	@Embedded
	private Delivery delivery = null;
	
	@Valid
	@Embedded
	private Billing billing = null;
	
	@Embedded
	@Deprecated
	private CreditCard creditCard = null;

	
	@ManyToOne(targetEntity = Currency.class)
	@JoinColumn(name = "CURRENCY_ID")
	private Currency currency;
	
	@Type(type="locale")  
	@Column (name ="LOCALE")
	private Locale locale; 
	

	@JsonIgnore
	@ManyToOne(targetEntity = MerchantStore.class)
	@JoinColumn(name="MERCHANTID")
	private MerchantStore merchant;
	
	//@OneToMany(mappedBy = "order")
	//private Set<OrderAccount> orderAccounts = new HashSet<OrderAccount>();
	
	@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
	private Set<OrderProduct> orderProducts = new LinkedHashSet<OrderProduct>();
	
	@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
	@OrderBy(clause = "sort_order asc")
	private Set<OrderTotal> orderTotal = new LinkedHashSet<OrderTotal>();
	
	@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
	@OrderBy(clause = "ORDER_STATUS_HISTORY_ID asc")
	private Set<OrderStatusHistory> orderHistory = new LinkedHashSet<OrderStatusHistory>();
	
	@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
	private Set<OrderAttribute> orderAttributes = new LinkedHashSet<OrderAttribute>();
	
	public Order() {
	}
	
	@Column (name ="CUSTOMER_EMAIL_ADDRESS", length=50, nullable=false)
	private String customerEmailAddress;


	@Override
	public Long getId() {
		return id;
	}
	
	@Override
	public void setId(Long id) {
		this.id = id;
	}

	public OrderStatus getStatus() {
		return status;
	}

	public void setStatus(OrderStatus status) {
		this.status = status;
	}

	public Date getLastModified() {
		return CloneUtils.clone(lastModified);
	}

	public void setLastModified(Date lastModified) {
		this.lastModified = CloneUtils.clone(lastModified);
	}

	public Date getDatePurchased() {
		return CloneUtils.clone(datePurchased);
	}

	public void setDatePurchased(Date datePurchased) {
		this.datePurchased = CloneUtils.clone(datePurchased);
	}

	public Date getOrderDateFinished() {
		return CloneUtils.clone(orderDateFinished);
	}

	public void setOrderDateFinished(Date orderDateFinished) {
		this.orderDateFinished = CloneUtils.clone(orderDateFinished);
	}

	public BigDecimal getCurrencyValue() {
		return currencyValue;
	}

	public void setCurrencyValue(BigDecimal currencyValue) {
		this.currencyValue = currencyValue;
	}

	public BigDecimal getTotal() {
		return total;
	}

	public void setTotal(BigDecimal total) {
		this.total = total;
	}


	public String getIpAddress() {
		return ipAddress;
	}

	public void setIpAddress(String ipAddress) {
		this.ipAddress = ipAddress;
	}


	public String getPaymentModuleCode() {
		return paymentModuleCode;
	}

	public void setPaymentModuleCode(String paymentModuleCode) {
		this.paymentModuleCode = paymentModuleCode;
	}



	public String getShippingModuleCode() {
		return shippingModuleCode;
	}

	public void setShippingModuleCode(String shippingModuleCode) {
		this.shippingModuleCode = shippingModuleCode;
	}



	public Currency getCurrency() {
		return currency;
	}

	public void setCurrency(Currency currency) {
		this.currency = currency;
	}

	public MerchantStore getMerchant() {
		return merchant;
	}

	public void setMerchant(MerchantStore merchant) {
		this.merchant = merchant;
	}

	public Set<OrderProduct> getOrderProducts() {
		return orderProducts;
	}

	public void setOrderProducts(Set<OrderProduct> orderProducts) {
		this.orderProducts = orderProducts;
	}

	public Set<OrderTotal> getOrderTotal() {
		return orderTotal;
	}

	public void setOrderTotal(Set<OrderTotal> orderTotal) {
		this.orderTotal = orderTotal;
	}

	public Set<OrderStatusHistory> getOrderHistory() {
		return orderHistory;
	}

	public void setOrderHistory(Set<OrderStatusHistory> orderHistory) {
		this.orderHistory = orderHistory;
	}


	public void setDelivery(Delivery delivery) {
		this.delivery = delivery;
	}

	public Delivery getDelivery() {
		return delivery;
	}

	public void setBilling(Billing billing) {
		this.billing = billing;
	}

	public Billing getBilling() {
		return billing;
	}

	public Long getCustomerId() {
		return customerId;
	}

	public void setCustomerId(Long customerId) {
		this.customerId = customerId;
	}
	

	public String getCustomerEmailAddress() {
		return customerEmailAddress;
	}

	public void setCustomerEmailAddress(String customerEmailAddress) {
		this.customerEmailAddress = customerEmailAddress;
	}


	public void setChannel(OrderChannel channel) {
		this.channel = channel;
	}


	public OrderChannel getChannel() {
		return channel;
	}


	public void setCreditCard(CreditCard creditCard) {
		this.creditCard = creditCard;
	}


	public CreditCard getCreditCard() {
		return creditCard;
	}


	public void setPaymentType(PaymentType paymentType) {
		this.paymentType = paymentType;
	}


	public PaymentType getPaymentType() {
		return paymentType;
	}
	
	public OrderType getOrderType() {
		return orderType;
	}

	public void setOrderType(OrderType orderType) {
		this.orderType = orderType;
	}
	
	public Locale getLocale() {
		return locale;
	}

	public void setLocale(Locale locale) {
		this.locale = locale;
	}
	
	public Boolean getCustomerAgreement() {
		return customerAgreement;
	}

	public void setCustomerAgreement(Boolean customerAgreement) {
		this.customerAgreement = customerAgreement;
	}

	public Boolean getConfirmedAddress() {
		return confirmedAddress;
	}

	public void setConfirmedAddress(Boolean confirmedAddress) {
		this.confirmedAddress = confirmedAddress;
	}
	
	public Set<OrderAttribute> getOrderAttributes() {
		return orderAttributes;
	}

	public void setOrderAttributes(Set<OrderAttribute> orderAttributes) {
		this.orderAttributes = orderAttributes;
	}
	
	public String getShoppingCartCode() {
		return shoppingCartCode;
	}

	public void setShoppingCartCode(String shoppingCartCode) {
		this.shoppingCartCode = shoppingCartCode;
	}

}


```
