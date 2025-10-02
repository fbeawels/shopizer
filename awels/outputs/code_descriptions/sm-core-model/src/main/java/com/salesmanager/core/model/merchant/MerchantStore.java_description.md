# MerchantStore.java

## Review

## 1. Summary  

`MerchantStore` is a JPA‑entity that models a merchant’s store in the SalesManager application.  
The class is persisted in the `MERCHANT_STORE` table and contains a wide range of attributes that describe a store’s identity, location, branding, and configuration (templates, currency, language, etc.).  

Key components  
| Component | Role |
|-----------|------|
| **Fields** | Persisted columns (e.g., `storename`, `code`, `country`, `currency`, …) and derived/auxiliary state (`lineage`, `dateBusinessSince`). |
| **JPA annotations** | Map the class to the database, define primary key strategy, relationships (ManyToOne, OneToMany, ManyToMany), and column constraints. |
| **Validation annotations** | Enforce basic constraints (`@NotEmpty`, `@Pattern`, `@Email`) at the persistence layer. |
| **AuditSection** | Embedded component that tracks creation/modification timestamps. |
| **Auditable interface** | Allows a generic audit API (via `getAuditSection()`/`setAuditSection()`). |
| **SalesManagerEntity base class** | Provides generic id handling, `clone()` support, and (presumably) common equals/hashCode implementations. |

Design patterns & libraries  
* **Data‑Access Object (DAO)/Repository** – The entity will be managed by Spring/Hibernate repositories.  
* **Builder / Parameterized constructors** – Small convenience constructors exist for quick instance creation.  
* **Jackson** – `@JsonIgnore` is used to avoid infinite recursion when serializing the entity.  
* **Java Bean Validation** – Constraint annotations from `javax.validation` enforce basic data integrity.  

---

## 2. Detailed Description  

### 2.1 Entity mapping  
* **Table** – `MERCHANT_STORE` with an index on `LINEAGE`.  
* **Primary key** – `MERCHANT_ID` generated via a table generator (`SM_SEQUENCER`).  
* **Embedded** – `AuditSection` holds audit fields (`created`, `modified`, etc.).  
* **Relationships**  
  * **Parent/Child** – `parent` (ManyToOne) and `stores` (OneToMany, cascade on remove).  
  * **Country / Zone** – `country` (mandatory) and `zone` (optional).  
  * **Currency** – mandatory ManyToOne.  
  * **Default & multiple languages** – `defaultLanguage` (mandatory) and `languages` (ManyToMany).  
* **Field constraints** – `@NotEmpty`, `@Pattern`, `@Email` provide basic validation.  

### 2.2 Runtime behavior  
* **Construction** – Default no‑arg constructor is required by JPA.  Two convenience constructors set the key properties.  
* **Date handling** – `inBusinessSince` is a `java.util.Date`; accessors clone the value (`CloneUtils.clone()`) to avoid exposing internal mutability.  
* **Derived state** – `dateBusinessSince` is transient and is meant for presentation only (e.g., formatted string).  
* **JSON serialization** – `@JsonIgnore` on navigation fields prevents circular references when converting to JSON.  

### 2.3 Assumptions & constraints  
* **JPA/Hibernate** – The persistence provider must support `@TableGenerator`.  
* **Validation framework** – Constraints rely on a Bean Validation provider (e.g., Hibernate Validator).  
* **Date API** – Uses legacy `Date`; expects callers to supply dates without time zone information.  
* **AuditSection** – The class assumes an embedded audit component that handles timestamps and user info.  

### 2.4 Architecture & design choices  
* **Separation of concerns** – Entity focuses on persistence mapping; business logic resides elsewhere (service layer).  
* **Immutability** – Only `inBusinessSince` is defensively copied; other fields are mutable but typical for entities.  
* **Embedded audit** – Keeps audit data close to the entity while still allowing a shared `AuditSection` implementation across models.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| **Constructors** | Create instances; set id, code, name (and optionally email). | `Integer id, String code, String name [, String storeEmailAddress]` | New `MerchantStore` | None |
| `setId(Integer)` / `getId()` | Implements `SalesManagerEntity`’s id contract. | `Integer id` | `Integer` | None |
| `setStorename(String)` / `getStorename()` | Store's display name. | `String` | `String` | None |
| `setStorephone(String)` / `getStorephone()` | Phone number. | `String` | `String` | None |
| `setStoreaddress(String)` / `getStoreaddress()` | Physical address. | `String` | `String` | None |
| `setStorecity(String)` / `getStorecity()` | City. | `String` | `String` | None |
| `setStorepostalcode(String)` / `getStorepostalcode()` | Postal code. | `String` | `String` | None |
| `setCountry(Country)` / `getCountry()` | Country reference. | `Country` | `Country` | None |
| `setZone(Zone)` / `getZone()` | Zone (state/province) reference. | `Zone` | `Zone` | None |
| `setStorestateprovince(String)` / `getStorestateprovince()` | State/Province name. | `String` | `String` | None |
| `setCurrency(Currency)` / `getCurrency()` | Currency reference. | `Currency` | `Currency` | None |
| `setWeightunitcode(String)` / `getWeightunitcode()` | Weight unit code (default `LB`). | `String` | `String` | None |
| `setSeizeunitcode(String)` / `getSeizeunitcode()` | Measure unit for size (default `IN`). | `String` | `String` | None |
| `getInBusinessSince()` / `setInBusinessSince(Date)` | Business start date (defensive copy). | `Date` | `Date` | Defensively clones input/output |
| `setDefaultLanguage(Language)` / `getDefaultLanguage()` | Default language. | `Language` | `Language` | None |
| `setLanguages(List<Language>)` / `getLanguages()` | Supported languages. | `List<Language>` | `List<Language>` | None |
| `setStoreLogo(String)` / `getStoreLogo()` | Logo file reference. | `String` | `String` | None |
| `setStoreTemplate(String)` / `getStoreTemplate()` | Store template identifier. | `String` | `String` | None |
| `setInvoiceTemplate(String)` / `getInvoiceTemplate()` | Invoice template identifier. | `String` | `String` | None |
| `setDomainName(String)` / `getDomainName()` | Domain name. | `String` | `String` | None |
| `setContinueshoppingurl(String)` / `getContinueshoppingurl()` | URL for continuing shopping. | `String` | `String` | None |
| `setStoreEmailAddress(String)` / `getStoreEmailAddress()` | Contact email. | `String` | `String` | None |
| `setDateBusinessSince(String)` / `getDateBusinessSince()` | Transient formatted date. | `String` | `String` | None |
| `setCurrencyFormatNational(boolean)` / `isCurrencyFormatNational()` | Flag whether to use national currency format. | `boolean` | `boolean` | None |
| `setAuditSection(AuditSection)` / `getAuditSection()` | Auditing integration. | `AuditSection` | `AuditSection` | None |
| `setParent(MerchantStore)` / `getParent()` | Hierarchical parent store. | `MerchantStore` | `MerchantStore` | None |
| `setStores(Set<MerchantStore>)` / `getStores()` | Child stores. | `Set<MerchantStore>` | `Set<MerchantStore>` | None |
| `isRetailer()` / `setRetailer(Boolean)` | Retailer flag. | `Boolean` | `Boolean` | None |

All setters follow a classic JavaBean pattern; getters return immutable copies where applicable. No business logic resides in this class; it is a pure persistence model.

---

## 4. Dependencies  

| Library / API | Type | Role |
|---------------|------|------|
| `javax.persistence.*` | JPA (Hibernate / EclipseLink) | ORM mapping, entity lifecycle |
| `javax.validation.*` | Bean Validation | Field constraints (`@NotEmpty`, `@Pattern`, `@Email`) |
| `com.fasterxml.jackson.annotation.JsonIgnore` | Jackson | JSON serialization control |
| `com.salesmanager.core.constants.MeasureUnit` | Project enum | Default weight/size units |
| `com.salesmanager.core.model.common.audit.*` | Project audit framework | Embedded audit section |
| `com.salesmanager.core.model.generic.SalesManagerEntity` | Project base class | Generic id handling, equals/hashCode |
| `com.salesmanager.core.model.reference.*` | Project reference entities | Country, Zone, Currency, Language |
| `com.salesmanager.core.utils.CloneUtils` | Project utility | Defensive cloning of `Date` |
| `java.util.*` | Java SE | Collections, Date |

All dependencies are either standard Java SE or project‑specific, with the exception of JPA and Jackson, which are common third‑party libraries.

---

## 5. Additional Notes  

### 5.1 Strengths  
* **Clear mapping** – The entity is well‑structured, with explicit column names and constraints.  
* **Audit integration** – Using an embedded `AuditSection` centralizes auditing logic.  
* **Defensive date handling** – Prevents callers from mutating internal state.  
* **Validation annotations** – Provide immediate feedback at the persistence layer.  

### 5.2 Potential Issues & Edge Cases  
1. **Mutable `Date`** – Even though cloned, `Date` is still mutable; consider switching to `java.time.LocalDate` or `java.time.Instant` to remove the need for defensive copying.  
2. **`lineage` field** – No business logic is present; ensure that the value is properly set/updated elsewhere (e.g., by a service).  
3. **`parent`/`stores` cycle** – JPA can handle bidirectional relationships, but serialization may still fail if not careful; the use of `@JsonIgnore` mitigates this.  
4. **Missing equals/hashCode** – The class relies on `SalesManagerEntity`; ensure that `equals` and `hashCode` are properly implemented there to avoid issues in collections.  
5. **Validation on `code`** – The regex allows only alphanumerics and underscore; if hyphens are needed, update the pattern.  
6. **`useCache` flag** – It is persisted but never read in this class; confirm that the flag is used consistently by consumers.  

### 5.3 Suggested Enhancements  
* **Builder pattern** – Provide a fluent builder for creating instances in a readable way.  
* **LocalDate/Instant** – Replace `Date` with the Java Time API for immutability and better time‑zone handling.  
* **Custom validation messages** – Add `message` attributes to annotations for clearer error reporting.  
* **Lombok integration** – Reduce boilerplate getters/setters and constructors (if project policy permits).  
* **Unit tests** – Verify that JSON serialization does not include navigational fields and that defensive copying works as expected.  

Overall, `MerchantStore` is a solid, well‑documented JPA entity that serves its purpose in the SalesManager domain. The main areas for improvement involve modernizing the date handling and tightening validation/message handling.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.merchant;

import java.util.ArrayList;
import java.util.Date;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.FetchType;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Index;
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
import javax.validation.constraints.Email;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.Pattern;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.core.constants.MeasureUnit;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.currency.Currency;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.core.utils.CloneUtils;

@Entity
@Table(name = "MERCHANT_STORE",
	indexes = @Index(columnList = "LINEAGE"))
public class MerchantStore extends SalesManagerEntity<Integer, MerchantStore> implements Auditable {

  private static final long serialVersionUID = 1L;

  public final static String DEFAULT_STORE = "DEFAULT";
  
  public MerchantStore(Integer id, String code, String name) {
	  this.id = id;
	  this.code = code;
	  this.storename = name;
	  
  }

  public MerchantStore(Integer id, String code, String name, String storeEmailAddress) {
    this.id = id;
    this.code = code;
    this.storename = name;
    this.storeEmailAddress = storeEmailAddress;
  }



	@Id
	@Column(name = "MERCHANT_ID", unique = true, nullable = false)
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT", pkColumnValue = "STORE_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Integer id;

	@Embedded
	private AuditSection auditSection = new AuditSection();

	@JsonIgnore
	@ManyToOne
	@JoinColumn(name = "PARENT_ID")
	private MerchantStore parent;

	@JsonIgnore
	@OneToMany(mappedBy = "parent", cascade = CascadeType.REMOVE)
	private Set<MerchantStore> stores = new HashSet<MerchantStore>();

	@Column(name = "IS_RETAILER")
	private Boolean retailer = false;

	@NotEmpty
	@Column(name = "STORE_NAME", nullable = false, length = 100)
	private String storename;

	@NotEmpty
	@Pattern(regexp = "^[a-zA-Z0-9_]*$")
	@Column(name = "STORE_CODE", nullable = false, unique = true, length = 100)
	private String code;
	
    @Column(name = "LINEAGE")
    private String lineage;

	@NotEmpty
	@Column(name = "STORE_PHONE", length = 50)
	private String storephone;

	@Column(name = "STORE_ADDRESS")
	private String storeaddress;

	@NotEmpty
	@Column(name = "STORE_CITY", length = 100)
	private String storecity;

	@NotEmpty
	@Column(name = "STORE_POSTAL_CODE", length = 15)
	private String storepostalcode;

	@JsonIgnore
	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Country.class)
	@JoinColumn(name = "COUNTRY_ID", nullable = false, updatable = true)
	private Country country;

	@JsonIgnore
	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Zone.class)
	@JoinColumn(name = "ZONE_ID", nullable = true, updatable = true)
	private Zone zone;

	@Column(name = "STORE_STATE_PROV", length = 100)
	private String storestateprovince;

	@Column(name = "WEIGHTUNITCODE", length = 5)
	private String weightunitcode = MeasureUnit.LB.name();

	@Column(name = "SEIZEUNITCODE", length = 5)
	private String seizeunitcode = MeasureUnit.IN.name();

	@Temporal(TemporalType.DATE)
	@Column(name = "IN_BUSINESS_SINCE")
	private Date inBusinessSince = new Date();

	@Transient
	private String dateBusinessSince;

	@JsonIgnore
	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Language.class)
	@JoinColumn(name = "LANGUAGE_ID", nullable = false)
	private Language defaultLanguage;

	@JsonIgnore
	@NotEmpty
	@ManyToMany(fetch = FetchType.LAZY)
	@JoinTable(name = "MERCHANT_LANGUAGE")
	private List<Language> languages = new ArrayList<Language>();

	@Column(name = "USE_CACHE")
	private boolean useCache = false;

	@Column(name = "STORE_TEMPLATE", length = 25)
	private String storeTemplate;

	@Column(name = "INVOICE_TEMPLATE", length = 25)
	private String invoiceTemplate;

	@Column(name = "DOMAIN_NAME", length = 80)
	private String domainName;

	@JsonIgnore
	@Column(name = "CONTINUESHOPPINGURL", length = 150)
	private String continueshoppingurl;

	@Email
	@NotEmpty
	@Column(name = "STORE_EMAIL", length = 60, nullable = false)
	private String storeEmailAddress;

	@JsonIgnore
	@Column(name = "STORE_LOGO", length = 100)
	private String storeLogo;

	@JsonIgnore
	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Currency.class)
	@JoinColumn(name = "CURRENCY_ID", nullable = false)
	private Currency currency;

	@Column(name = "CURRENCY_FORMAT_NATIONAL")
	private boolean currencyFormatNational;

	public MerchantStore() {
	}

	public boolean isUseCache() {
		return useCache;
	}

	public void setUseCache(boolean useCache) {
		this.useCache = useCache;
	}

	@Override
	public void setId(Integer id) {
		this.id = id;
	}

	@Override
	public Integer getId() {
		return this.id;
	}

	public String getStorename() {
		return storename;
	}

	public void setStorename(String storename) {
		this.storename = storename;
	}

	public String getStorephone() {
		return storephone;
	}

	public void setStorephone(String storephone) {
		this.storephone = storephone;
	}

	public String getStoreaddress() {
		return storeaddress;
	}

	public void setStoreaddress(String storeaddress) {
		this.storeaddress = storeaddress;
	}

	public String getStorecity() {
		return storecity;
	}

	public void setStorecity(String storecity) {
		this.storecity = storecity;
	}

	public String getStorepostalcode() {
		return storepostalcode;
	}

	public void setStorepostalcode(String storepostalcode) {
		this.storepostalcode = storepostalcode;
	}

	public Country getCountry() {
		return country;
	}

	public void setCountry(Country country) {
		this.country = country;
	}

	public Zone getZone() {
		return zone;
	}

	public void setZone(Zone zone) {
		this.zone = zone;
	}

	public String getStorestateprovince() {
		return storestateprovince;
	}

	public void setStorestateprovince(String storestateprovince) {
		this.storestateprovince = storestateprovince;
	}

	public Currency getCurrency() {
		return currency;
	}

	public void setCurrency(Currency currency) {
		this.currency = currency;
	}

	public String getWeightunitcode() {
		return weightunitcode;
	}

	public void setWeightunitcode(String weightunitcode) {
		this.weightunitcode = weightunitcode;
	}

	public String getSeizeunitcode() {
		return seizeunitcode;
	}

	public void setSeizeunitcode(String seizeunitcode) {
		this.seizeunitcode = seizeunitcode;
	}

	public Date getInBusinessSince() {
		return CloneUtils.clone(inBusinessSince);
	}

	public void setInBusinessSince(Date inBusinessSince) {
		this.inBusinessSince = CloneUtils.clone(inBusinessSince);
	}

	public Language getDefaultLanguage() {
		return defaultLanguage;
	}

	public void setDefaultLanguage(Language defaultLanguage) {
		this.defaultLanguage = defaultLanguage;
	}

	public List<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(List<Language> languages) {
		this.languages = languages;
	}

	public String getStoreLogo() {
		return storeLogo;
	}

	public void setStoreLogo(String storeLogo) {
		this.storeLogo = storeLogo;
	}

	public String getStoreTemplate() {
		return storeTemplate;
	}

	public void setStoreTemplate(String storeTemplate) {
		this.storeTemplate = storeTemplate;
	}

	public String getInvoiceTemplate() {
		return invoiceTemplate;
	}

	public void setInvoiceTemplate(String invoiceTemplate) {
		this.invoiceTemplate = invoiceTemplate;
	}

	public String getDomainName() {
		return domainName;
	}

	public void setDomainName(String domainName) {
		this.domainName = domainName;
	}

	public String getContinueshoppingurl() {
		return continueshoppingurl;
	}

	public void setContinueshoppingurl(String continueshoppingurl) {
		this.continueshoppingurl = continueshoppingurl;
	}

	public String getStoreEmailAddress() {
		return storeEmailAddress;
	}

	public void setStoreEmailAddress(String storeEmailAddress) {
		this.storeEmailAddress = storeEmailAddress;
	}

	public String getCode() {
		return code;
	}

	public void setCode(String code) {
		this.code = code;
	}

	public void setDateBusinessSince(String dateBusinessSince) {
		this.dateBusinessSince = dateBusinessSince;
	}

	public String getDateBusinessSince() {
		return dateBusinessSince;
	}

	public void setCurrencyFormatNational(boolean currencyFormatNational) {
		this.currencyFormatNational = currencyFormatNational;
	}

	public boolean isCurrencyFormatNational() {
		return currencyFormatNational;
	}

	@Override
	public AuditSection getAuditSection() {
		return this.auditSection;
	}

	@Override
	public void setAuditSection(AuditSection audit) {
		this.auditSection = audit;

	}

	public MerchantStore getParent() {
		return parent;
	}

	public void setParent(MerchantStore parent) {
		this.parent = parent;
	}

	public Set<MerchantStore> getStores() {
		return stores;
	}

	public void setStores(Set<MerchantStore> stores) {
		this.stores = stores;
	}

	public Boolean isRetailer() {
		return retailer;
	}


	public void setRetailer(Boolean retailer) {
		this.retailer = retailer;
	}

}



```
