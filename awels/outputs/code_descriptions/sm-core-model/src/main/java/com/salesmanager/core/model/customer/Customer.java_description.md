# Customer.java

## Review

## 1. Summary  
The **`Customer`** class is a JPA entity that represents a customer in a multi‑store e‑commerce platform.  
It is mapped to the `CUSTOMER` table and carries a wealth of information – authentication credentials, personal data, address information, reviews, and membership groups.  

Key responsibilities:

| Component | Role |
|-----------|------|
| `@Entity`, `@Table` | Persist to DB |
| `AuditSection` | Auditing (created/updated timestamps) |
| `@Embedded` fields (`Delivery`, `Billing`, `CredentialsReset`, `AuditSection`) | Composite value objects |
| Collections (`attributes`, `reviews`, `groups`) | Relationships with other entities |
| Validation annotations (`@Email`, `@NotEmpty`) | Enforce data integrity at the bean level |
| `@JsonIgnore` | Hide sensitive fields from JSON serialization |
| `CloneUtils` | Defensive copying of mutable `Date` objects |

The class relies heavily on standard JPA annotations, Hibernate extensions, Jackson for JSON control, and a handful of domain‑specific types (`SalesManagerEntity`, `AuditSection`, `CustomerGender`, etc.). It implements `Auditable` to expose the embedded audit section.

---

## 2. Detailed Description  

### 2.1 Table & Constraints  
```java
@Table(name = "CUSTOMER",
       uniqueConstraints=@UniqueConstraint(columnNames = {"MERCHANT_ID", "CUSTOMER_NICK"}))
```
A composite unique key guarantees that a customer’s `nick` is unique only within a particular store (`MERCHANT_ID`).

### 2.2 Primary Key Generation  
```java
@Id
@TableGenerator(name="TABLE_GEN", table="SM_SEQUENCER",
                pkColumnName="SEQ_NAME", valueColumnName="SEQ_COUNT",
                pkColumnValue="CUSTOMER_SEQ_NEXT_VAL")
@GeneratedValue(strategy=GenerationType.TABLE, generator="TABLE_GEN")
```
Uses a table‑based sequence generator (`SM_SEQUENCER`). Good for database portability but can become a bottleneck under high‑concurrency scenarios.

### 2.3 Relationships  

| Relationship | Mapping | Cascade / Fetch | Notes |
|--------------|---------|-----------------|-------|
| `attributes` | `@OneToMany(mappedBy="customer")` | LAZY, ALL | Bidirectional (the `CustomerAttribute` side owns the FK) |
| `reviews` | `@OneToMany(mappedBy="customer")` | LAZY (default) | No cascade – reviews are managed independently |
| `merchantStore` | `@ManyToOne` | LAZY | Store must exist; `MERCHANT_ID` FK |
| `defaultLanguage` | `@ManyToOne` | LAZY | Non‑nullable |
| `groups` | `@ManyToMany` | LAZY, REFRESH | Join table `CUSTOMER_GROUP`; Hibernate cascade types added manually (DETACH, LOCK, REFRESH, REPLICATE). The use of `List<Group>` may lead to duplicate entries; a `Set` is typically safer. |

### 2.4 Embedded Components  
- `Delivery`, `Billing`, `CredentialsReset`, `AuditSection` are embedded value objects.  
- `@Valid` on `Billing` ensures nested validation (e.g., credit card number) when persisting the Customer.

### 2.5 Defensive Copying  
`Date` is mutable; `getDateOfBirth()` and `setDateOfBirth()` use `CloneUtils.clone()` to avoid exposing internal state.

### 2.6 JSON Control  
Sensitive fields (`password`, audit section, group list, transient state lists) are annotated with `@JsonIgnore` to prevent accidental exposure in REST responses.

### 2.7 Transient State Lists  
`showCustomerStateList`, `showBillingStateList`, `showDeliveryStateList` are marked `@Transient` and ignored by JSON. They appear to be UI‑helper fields (e.g., populating dropdowns). As they are not persisted, they have no impact on the database schema.

---

## 3. Functions/Methods  

| Method | Purpose | Input | Output | Side‑effects |
|--------|---------|-------|--------|--------------|
| `getId() / setId(Long)` | Primary key getter/setter | – | `Long` | – |
| `getDateOfBirth() / setDateOfBirth(Date)` | Access DOB defensively | `Date` | `Date` | Defensive clone |
| `getEmailAddress() / setEmailAddress(String)` | Contact email | – | – | – |
| `getNick() / setNick(String)` | Unique username per store | – | – | – |
| `getCompany() / setCompany(String)` | Company name | – | – | – |
| `getPassword() / setPassword(String)` | Encrypted password | – | – | – |
| `isAnonymous() / setAnonymous(boolean)` | Flag for anonymous visitors | – | – | – |
| `getReviews() / setReviews(List<ProductReview>)` | Customer reviews | – | – | – |
| `getMerchantStore() / setMerchantStore(MerchantStore)` | Store association | – | – | – |
| `getDelivery() / setDelivery(Delivery)` | Delivery address | – | – | – |
| `getBilling() / setBilling(Billing)` | Billing address | – | – | – |
| `getGroups() / setGroups(List<Group>)` | Membership groups | – | – | – |
| `getShowCustomerStateList() / setShowCustomerStateList(String)` | UI helper | – | – | – |
| `getDefaultLanguage() / setDefaultLanguage(Language)` | Preferred language | – | – | – |
| `getAttributes() / setAttributes(Set<CustomerAttribute>)` | Custom attributes | – | – | – |
| `getGender() / setGender(CustomerGender)` | Customer gender | – | – | – |
| `getCustomerReviewAvg() / setCustomerReviewAvg(BigDecimal)` | Average rating | – | – | – |
| `getCustomerReviewCount() / setCustomerReviewCount(Integer)` | Number of reviews | – | – | – |
| `getAuditSection() / setAuditSection(AuditSection)` | Auditing | – | – | – |
| `getProvider() / setProvider(String)` | External auth provider | – | – | – |
| `getCredentialsResetRequest() / setCredentialsResetRequest(CredentialsReset)` | Password reset data | – | – | – |

All setters perform direct assignment; getters return the stored value (with cloning for `Date`). No complex business logic is present.

---

## 4. Dependencies  

| Category | Library / Framework | Notes |
|----------|---------------------|-------|
| **Persistence** | `javax.persistence` (JPA) | Entity mapping, relationships |
| **Validation** | `javax.validation` (JSR‑303) | `@Email`, `@NotEmpty`, `@Valid` |
| **ORM Enhancements** | `org.hibernate.annotations` | Custom cascade types |
| **JSON** | `com.fasterxml.jackson.annotation.JsonIgnore` | Hide sensitive fields |
| **Domain** | `com.salesmanager.core.*` | `SalesManagerEntity`, `AuditSection`, `CustomerGender`, etc. |
| **Utility** | `com.salesmanager.core.utils.CloneUtils` | Defensive copying of `Date` |
| **JPA / Hibernate** | `org.hibernate` (via JPA provider) | Entity persistence |
| **Others** | `java.util`, `java.math` | Standard JDK classes |

All dependencies are either standard JDK or widely‑used open‑source libraries (Hibernate, Jackson, Bean Validation). No platform‑specific (e.g., Oracle‑only) dependencies are evident.

---

## 5. Additional Notes  

### 5.1 Strengths  
- **Clear separation of concerns** – value objects (`Billing`, `Delivery`) are embedded, reducing complexity.  
- **Auditability** – embedded `AuditSection` supports created/modified timestamps automatically.  
- **Defensive copying** of mutable `Date` fields protects internal state.  
- **JSON safety** – sensitive data is masked.

### 5.2 Potential Improvements & Edge Cases  

| Area | Issue | Suggested Fix |
|------|-------|---------------|
| **Primary key generation** | Table‑based generator can become a contention point under heavy write load. | Consider `GenerationType.IDENTITY` or a native sequence if the DB supports it. |
| **Collection types** | `List<Group>` + `List<ProductReview>` can allow duplicate entries and are less efficient for lookups. | Switch to `Set<Group>` and `Set<ProductReview>` or enforce uniqueness in business logic. |
| **Cascade configuration** | Mixing JPA `cascade = CascadeType.REFRESH` with Hibernate `CascadeType` is redundant and can be confusing. | Keep a single cascade strategy (preferably `CascadeType.ALL` or `REFRESH` only if needed). |
| **Equals / hashCode** | Entity lacks `equals()` and `hashCode()`. | Implement based on immutable fields (`id` or business key) to avoid collection issues. |
| **Email uniqueness** | Unique constraint on `CUSTOMER_NICK` only. | Add a DB index or unique constraint on `CUSTOMER_EMAIL_ADDRESS` per store if required. |
| **Password storage** | Stored as plain string. | Ensure password is hashed before persisting and never returned in getters. |
| **Date handling** | Uses legacy `Date`. | Migrate to `java.time.LocalDate` / `LocalDateTime` for better immutability. |
| **Transient UI helpers** | `showCustomerStateList*` are not persisted but may leak state across requests. | Keep these in a separate view‑model rather than the entity. |
| **Validation** | No `@NotNull` on required fields (`emailAddress`, `merchantStore`). | Add explicit constraints to enforce business rules at the persistence layer. |
| **AuditSection** | No `@JsonIgnore` – might leak audit data. | Annotate with `@JsonIgnore` or expose via a DTO. |

### 5.3 Future Enhancements  
1. **DTO layer** – expose a customer DTO for API responses, shielding the entity from serialization concerns.  
2. **Soft delete** – add a `deleted` flag and adjust queries accordingly.  
3. **Address reuse** – extract a common `Address` embeddable for `Delivery` and `Billing`.  
4. **Event sourcing** – publish domain events when a customer is created/updated.  
5. **Security** – integrate Spring Security’s `UserDetails` for authentication integration.  

---

### Verdict  
The `Customer` entity is well‑structured and follows common JPA best practices. It encapsulates business data, enforces validation, and provides a clean API for persistence. Minor refactoring (collection types, cascade handling, equals/hashCode, date handling) and a few architectural enhancements (DTOs, soft delete) would further improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.customer;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.EnumType;
import javax.persistence.Enumerated;
import javax.persistence.FetchType;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.JoinColumn;
import javax.persistence.JoinTable;
import javax.persistence.ManyToMany;
import javax.persistence.ManyToOne;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.persistence.Transient;
import javax.persistence.UniqueConstraint;
import javax.validation.Valid;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotEmpty;

import org.hibernate.annotations.Cascade;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.core.constants.SchemaConstant;
import com.salesmanager.core.model.catalog.product.review.ProductReview;
import com.salesmanager.core.model.common.Billing;
import com.salesmanager.core.model.common.CredentialsReset;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.customer.attribute.CustomerAttribute;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.user.Group;
import com.salesmanager.core.utils.CloneUtils;

@Entity
@Table(name = "CUSTOMER", 
	 uniqueConstraints=
			@UniqueConstraint(columnNames = {"MERCHANT_ID", "CUSTOMER_NICK"}))
public class Customer extends SalesManagerEntity<Long, Customer> implements Auditable {
	private static final long serialVersionUID = 1L;
	
	@Id
	@Column(name = "CUSTOMER_ID", unique=true, nullable=false)
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT",
	pkColumnValue = "CUSTOMER_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;
	
	@JsonIgnore
	@Embedded
	private AuditSection auditSection = new AuditSection();
	
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "customer")
	private Set<CustomerAttribute> attributes = new HashSet<CustomerAttribute>();
	
	@Column(name="CUSTOMER_GENDER", length=1, nullable=true)
	@Enumerated(value = EnumType.STRING)
	private CustomerGender gender;


	@Temporal(TemporalType.TIMESTAMP)
	@Column(name="CUSTOMER_DOB")
	private Date dateOfBirth;
	
	@Email
	@NotEmpty
	@Column(name="CUSTOMER_EMAIL_ADDRESS", length=96, nullable=false)
	private String emailAddress;
	
	@Column(name="CUSTOMER_NICK", length=96)
	private String nick;// unique username per store

	@Column(name="CUSTOMER_COMPANY", length=100)
	private String company;
	
	@JsonIgnore
	@Column(name="CUSTOMER_PASSWORD", length=60)
	private String password;

	@Column(name="CUSTOMER_ANONYMOUS")
	private boolean anonymous;
	
	@Column(name = "REVIEW_AVG")
	private BigDecimal customerReviewAvg;

	@Column(name = "REVIEW_COUNT")
	private Integer customerReviewCount;
	
	@Column(name="PROVIDER")
	private String provider;
	

	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Language.class)
	@JoinColumn(name = "LANGUAGE_ID", nullable=false)
	private Language defaultLanguage;
	

	@OneToMany(mappedBy = "customer", targetEntity = ProductReview.class)
	private List<ProductReview> reviews = new ArrayList<ProductReview>();
	
	@JsonIgnore
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name="MERCHANT_ID", nullable=false)
	private MerchantStore merchantStore;
	

	@Embedded
	private Delivery delivery = null;
	
	@Valid
	@Embedded
	private Billing billing = null;
	
	@JsonIgnore
	@ManyToMany(fetch=FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinTable(name = "CUSTOMER_GROUP", joinColumns = { 
			@JoinColumn(name = "CUSTOMER_ID", nullable = false, updatable = false) }
			, 
			inverseJoinColumns = { @JoinColumn(name = "GROUP_ID", 
					nullable = false, updatable = false) }
	)
	@Cascade({
		org.hibernate.annotations.CascadeType.DETACH,
		org.hibernate.annotations.CascadeType.LOCK,
		org.hibernate.annotations.CascadeType.REFRESH,
		org.hibernate.annotations.CascadeType.REPLICATE
		
	})
	private List<Group> groups = new ArrayList<Group>();
	
	@JsonIgnore
	@Transient
	private String showCustomerStateList;
	
	@JsonIgnore
	@Transient
	private String showBillingStateList;
	
	@JsonIgnore
	@Transient
	private String showDeliveryStateList;

	@Embedded
	private CredentialsReset credentialsResetRequest = null;

	public Customer() {
	}

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}



	public Date getDateOfBirth() {
		return CloneUtils.clone(dateOfBirth);
	}

	public void setDateOfBirth(Date dateOfBirth) {
		this.dateOfBirth = CloneUtils.clone(dateOfBirth);
	}

	public String getEmailAddress() {
		return emailAddress;
	}

	public void setEmailAddress(String emailAddress) {
		this.emailAddress = emailAddress;
	}

	public String getNick() {
		return nick;
	}

	public void setNick(String nick) {
		this.nick = nick;
	}

	public String getCompany() {
		return company;
	}

	public void setCompany(String company) {
		this.company = company;
	}



	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}



	public boolean isAnonymous() {
		return anonymous;
	}

	public void setAnonymous(boolean anonymous) {
		this.anonymous = anonymous;
	}


	public List<ProductReview> getReviews() {
		return reviews;
	}

	public void setReviews(List<ProductReview> reviews) {
		this.reviews = reviews;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
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

	public void setGroups(List<Group> groups) {
		this.groups = groups;
	}

	public List<Group> getGroups() {
		return groups;
	}
	public String getShowCustomerStateList() {
		return showCustomerStateList;
	}

	public void setShowCustomerStateList(String showCustomerStateList) {
		this.showCustomerStateList = showCustomerStateList;
	}

	public String getShowBillingStateList() {
		return showBillingStateList;
	}

	public void setShowBillingStateList(String showBillingStateList) {
		this.showBillingStateList = showBillingStateList;
	}

	public String getShowDeliveryStateList() {
		return showDeliveryStateList;
	}

	public void setShowDeliveryStateList(String showDeliveryStateList) {
		this.showDeliveryStateList = showDeliveryStateList;
	}
	
	public Language getDefaultLanguage() {
		return defaultLanguage;
	}

	public void setDefaultLanguage(Language defaultLanguage) {
		this.defaultLanguage = defaultLanguage;
	}

	public void setAttributes(Set<CustomerAttribute> attributes) {
		this.attributes = attributes;
	}

	public Set<CustomerAttribute> getAttributes() {
		return attributes;
	}

	public void setGender(CustomerGender gender) {
		this.gender = gender;
	}

	public CustomerGender getGender() {
		return gender;
	}

	public BigDecimal getCustomerReviewAvg() {
		return customerReviewAvg;
	}

	public void setCustomerReviewAvg(BigDecimal customerReviewAvg) {
		this.customerReviewAvg = customerReviewAvg;
	}

	public Integer getCustomerReviewCount() {
		return customerReviewCount;
	}

	public void setCustomerReviewCount(Integer customerReviewCount) {
		this.customerReviewCount = customerReviewCount;
	}

	@Override
	public AuditSection getAuditSection() {
		return auditSection;
	}

	@Override
	public void setAuditSection(AuditSection auditSection) {
		this.auditSection = auditSection;
	}
	
	public String getProvider() {
		return provider;
	}

	public void setProvider(String provider) {
		this.provider = provider;
	}

	public CredentialsReset getCredentialsResetRequest() {
		return credentialsResetRequest;
	}

	public void setCredentialsResetRequest(CredentialsReset credentialsResetRequest) {
		this.credentialsResetRequest = credentialsResetRequest;
	}
	
}



```
