# ProductAvailability.java

## Review

## 1. Summary

| Item | Description |
|------|-------------|
| **Purpose** | `ProductAvailability` is a JPA entity that represents the stock‑and‑shipping information for a product (or a specific variant) in a particular store, region, or region‑variant. |
| **Key Components** | • **Identifiers** – `id` (auto‑generated) <br>• **Relations** – `product`, `merchantStore`, `productVariant`, and a collection of `ProductPrice`. <br>• **Attributes** – quantity, availability dates, region settings, SKU, dimensions, status flags, and price limits. <br>• **Audit** – embedded `AuditSection`. |
| **Frameworks/Libraries** | • Hibernate / JPA annotations (javax.persistence) <br>• Bean Validation (javax.validation) <br>• Jackson (`@JsonIgnore`) <br>• Custom utilities (`CloneUtils`) and domain classes from the SalesManager project. |

The entity is designed to enforce a unique key on `(MERCHANT_ID, PRODUCT_ID, PRODUCT_VARIANT, REGION_VARIANT)` and provide indexed queries on product and store.

---

## 2. Detailed Description

### 2.1 Architecture & Design

* **Persistence Layer** – The class is a standard JPA entity with a table generator strategy. All fields are mapped to columns, except the transient `defaultPrice()` method.
* **Relationships** – Many‑to‑one relationships are lazy loaded (`product`, `merchantStore`, `productVariant`). The `prices` collection is a one‑to‑many with cascade‑ALL, meaning that CRUD operations on `ProductAvailability` automatically propagate to its prices.
* **Audit** – `AuditSection` is embedded and managed via the `Auditable` interface, allowing automatic population of created/modified timestamps.
* **Validation** – The SKU field is validated with a regex to restrict allowed characters; the quantity field is `@NotNull` and defaults to `0`.

### 2.2 Execution Flow

| Phase | What Happens |
|-------|--------------|
| **Construction** | Two constructors are provided: a no‑arg constructor (required by JPA) and a convenience constructor that sets `product` and `merchantStore`. |
| **Persisting** | When a new instance is persisted, the table generator produces a unique ID. Relationships are resolved via foreign keys; the `prices` collection is persisted automatically due to cascade. |
| **Retrieval** | Lazy loading ensures that heavy relationships (`product`, `merchantStore`, etc.) are fetched only when accessed. The transient `defaultPrice()` method computes the default price on demand. |
| **Updating** | Setters modify the fields; JPA will detect changes and generate the appropriate SQL. The clone utilities guard against exposing mutable `Date` objects. |
| **Deletion** | Cascade removes associated `ProductPrice` records, while the owning side (product/merchant) must be handled separately. |

### 2.3 Assumptions & Constraints

* **Database** – Assumes a relational DB supporting sequences / table generators. The unique constraint does not explicitly forbid `NULL` values for `MERCHANT_ID`, `PRODUCT_VARIANT`, or `REGION_VARIANT`, which may lead to multiple rows with the same product but different optional columns.
* **Business Rules** – No explicit validation of quantity limits or region logic; the entity relies on application‑level checks.
* **Performance** – Lazy fetching reduces overhead but may lead to N+1 query problems if not managed carefully.

---

## 3. Functions / Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| **Constructors**<br>`ProductAvailability()` | JPA requirement | – | New empty instance | – |
| | `ProductAvailability(Product, MerchantStore)` | `Product`, `MerchantStore` | New instance with fields set | – |
| **Getters / Setters** | Standard property accessors for all mapped fields. | – | Property value | May trigger JPA dirty checking |
| `getProductDateAvailable()` / `setProductDateAvailable(Date)` | Return/clone a defensive copy of `productDateAvailable`. | `Date` (for setter) | Cloned `Date` | Avoids exposing mutable internal state |
| **`defaultPrice()`** | Returns the first `ProductPrice` marked as default; otherwise creates a new, transient `ProductPrice`. | – | `ProductPrice` | Creates a new object if none exist |
| **Auditable**<br>`getAuditSection()` / `setAuditSection()` | Accessor for embedded audit section. | – | `AuditSection` | – |
| **`getId()` / `setId(Long)`** | Implements `SalesManagerEntity` contract. | – | `Long` | – |

### Reusable / Utility Methods
* `defaultPrice()` can be reused across service layers to quickly fetch the price to display.
* The clone pattern used in the date getters can be copied for other mutable fields if added later.

---

## 4. Dependencies

| Dependency | Type | Role |
|------------|------|------|
| `javax.persistence.*` | JPA/Hibernate | Entity mapping, relationships, lifecycle |
| `javax.validation.constraints.*` | Bean Validation | SKU regex, non‑null quantity |
| `com.fasterxml.jackson.annotation.JsonIgnore` | Jackson | Hide sensitive relations from JSON |
| `com.salesmanager.core.constants.SchemaConstant` | Project constant | Default region value |
| `com.salesmanager.core.model.catalog.product.*` | Domain | Product, dimensions, price, variant |
| `com.salesmanager.core.model.common.audit.*` | Domain | Auditing section |
| `com.salesmanager.core.model.generic.SalesManagerEntity` | Domain | Base entity abstraction |
| `com.salesmanager.core.utils.CloneUtils` | Utility | Defensive cloning of `Date` |

All are either standard Java EE/Jakarta EE components or internal project libraries. No external runtime dependencies beyond the JPA provider.

---

## 5. Additional Notes & Recommendations

### 5.1 Potential Issues

1. **`defaultPrice()` Side‑Effect**  
   *If no default price exists, the method returns a brand‑new `ProductPrice` that isn’t persisted or linked to this `ProductAvailability`. Consumers of this method might mistakenly treat it as a real price, leading to inconsistencies or accidental inserts.*

2. **Unique Constraint with `NULL` Columns**  
   *Because `MERCHANT_ID`, `PRODUCT_VARIANT`, and `REGION_VARIANT` can be `NULL`, the DB will treat each `NULL` as distinct. Thus, the same product could have multiple rows with `NULL` merchant or variant, violating the intended uniqueness.*

3. **Missing `equals` / `hashCode`**  
   *The entity inherits from `SalesManagerEntity`, but if that base class does not override these methods, equality checks on detached entities (e.g., in collections) may behave unexpectedly.*

4. **Date Handling**  
   *Using `java.util.Date` is legacy. Switching to `java.time.LocalDate` (or `ZonedDateTime`) would simplify cloning and avoid time‑zone issues.*

5. **Audit Section Exposure**  
   *The `auditSection` is exposed via getters/setters. If the audit fields are modified outside the persistence context, audit consistency may be broken.*

### 5.2 Edge Cases

| Scenario | Current Handling | Suggested |
|----------|------------------|-----------|
| Product with **no prices** | `defaultPrice()` returns a new price object | Return `Optional<ProductPrice>` or `null` and document behavior |
| Product with **quantity zero** but `available = true` | No check; could mislead UI | Enforce `quantity > 0` if `available` is true |
| **Concurrent updates** | Not addressed | Add optimistic locking (`@Version`) or handle at service layer |

### 5.3 Future Enhancements

| Feature | Rationale | Implementation Notes |
|---------|-----------|----------------------|
| **Optimistic Locking** | Prevent lost updates when multiple threads modify availability | Add `@Version` field and handle `OptimisticLockException` |
| **Region/Sub‑region Hierarchy** | Support more granular shipping rules | Replace `region`/`regionVariant` strings with a dedicated `Region` entity |
| **Price Default Enforcement** | Guarantee exactly one default price | Add a unique constraint on `prices` table for the default flag, or enforce in service layer |
| **Soft Delete** | Preserve history | Add `isDeleted` flag and filter queries accordingly |
| **Metrics / Event Sourcing** | Track inventory changes | Emit events on quantity or price changes |
| **DTO / Mapper Layer** | Separate persistence model from API | Use MapStruct or similar to convert between entities and DTOs |

### 5.4 Code‑Quality Improvements

* **Use `@JsonIgnoreProperties`** to prevent lazy proxies from being serialized.
* **Add `@Column(length = 64)`** to `sku` for consistency.
* **Document contract of `defaultPrice()`** clearly in JavaDoc.
* **Consider `@Access(AccessType.FIELD)`** if you want to avoid method‑level proxies.
* **Use Lombok** (if allowed) to reduce boilerplate getters/setters.

---

### Final Verdict

`ProductAvailability` is a solid, well‑structured JPA entity that captures the necessary stock and shipping details for products in an e‑commerce context. The design leverages standard annotations and follows common enterprise patterns. Addressing the highlighted edge cases and adopting modern date/time APIs would strengthen the code’s robustness and future‑proof it against common pitfalls.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.catalog.product.availability;

import java.util.Date;
import java.util.HashSet;
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
import javax.persistence.ManyToOne;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.persistence.Transient;
import javax.persistence.UniqueConstraint;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.core.constants.SchemaConstant;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.ProductDimensions;
import com.salesmanager.core.model.catalog.product.price.ProductPrice;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.utils.CloneUtils;

@Entity
@Table(name = "PRODUCT_AVAILABILITY",
uniqueConstraints= @UniqueConstraint(columnNames = {"MERCHANT_ID", "PRODUCT_ID", "PRODUCT_VARIANT", "REGION_VARIANT"}),
indexes = 
	{ 
		@Index(name="PRD_AVAIL_STORE_PRD_IDX", columnList = "PRODUCT_ID,MERCHANT_ID"),
		@Index(name="PRD_AVAIL_PRD_IDX", columnList = "PRODUCT_ID")
	}
)

/**
 * Default availability
 * 
 * store
 * product id
 * 
 * variant null
 * regionVariant null
 * 
 * @author carlsamson
 *
 */
public class ProductAvailability extends SalesManagerEntity<Long, ProductAvailability> implements Auditable {

	/**
	* 
	*/
	private static final long serialVersionUID = 1L;

	@Embedded
	private AuditSection auditSection = new AuditSection();

	@Id
	@Column(name = "PRODUCT_AVAIL_ID", unique = true, nullable = false)
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT", pkColumnValue = "PRODUCT_AVAIL_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;

	@JsonIgnore
	@ManyToOne(targetEntity = Product.class)
	@JoinColumn(name = "PRODUCT_ID", nullable = false)
	private Product product;

	/** Specific retailer store **/
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "MERCHANT_ID", nullable = true)
	private MerchantStore merchantStore;
	
	/**
	 * This describes the availability of a product variant
	 */
	@ManyToOne(targetEntity = ProductVariant.class)
	@JoinColumn(name = "PRODUCT_VARIANT", nullable = true)
	private ProductVariant productVariant;
	
	@Pattern(regexp="^[a-zA-Z0-9_]*$")
	@Column(name = "SKU", nullable = true)
	private String sku;

	@Embedded
	private ProductDimensions dimensions;

	@NotNull
	@Column(name = "QUANTITY")
	private Integer productQuantity = 0;

	@Temporal(TemporalType.DATE)
	@Column(name = "DATE_AVAILABLE")
	private Date productDateAvailable;

	@Column(name = "REGION")
	private String region = SchemaConstant.ALL_REGIONS;

	@Column(name = "REGION_VARIANT")
	private String regionVariant;

	@Column(name = "OWNER")
	private String owner;

	@Column(name = "STATUS")
	private boolean productStatus = true; //can be used as flag for variant can be purchase or not

	@Column(name = "FREE_SHIPPING")
	private boolean productIsAlwaysFreeShipping;

	@Column(name = "AVAILABLE")
	private Boolean available;

	@Column(name = "QUANTITY_ORD_MIN")
	private Integer productQuantityOrderMin = 0;

	@Column(name = "QUANTITY_ORD_MAX")
	private Integer productQuantityOrderMax = 0;

	@OneToMany(fetch = FetchType.LAZY, mappedBy = "productAvailability", cascade = CascadeType.ALL)
	private Set<ProductPrice> prices = new HashSet<ProductPrice>();
	

	@Transient
	public ProductPrice defaultPrice() {
		for (ProductPrice price : prices) {
			if (price.isDefaultPrice()) {
				return price;
			}
		}
		return new ProductPrice();
	}

	public ProductAvailability() {
	}

	public ProductAvailability(Product product, MerchantStore store) {
		this.product = product;
		this.merchantStore = store;
	}

	public Integer getProductQuantity() {
		return productQuantity;
	}

	public void setProductQuantity(Integer productQuantity) {
		this.productQuantity = productQuantity;
	}

	public Date getProductDateAvailable() {
		return CloneUtils.clone(productDateAvailable);
	}

	public void setProductDateAvailable(Date productDateAvailable) {
		this.productDateAvailable = CloneUtils.clone(productDateAvailable);
	}

	public String getRegion() {
		return region;
	}

	public void setRegion(String region) {
		this.region = region;
	}

	public String getRegionVariant() {
		return regionVariant;
	}

	public void setRegionVariant(String regionVariant) {
		this.regionVariant = regionVariant;
	}

	public boolean getProductStatus() {
		return productStatus;
	}

	public void setProductStatus(boolean productStatus) {
		this.productStatus = productStatus;
	}

	public boolean getProductIsAlwaysFreeShipping() {
		return productIsAlwaysFreeShipping;
	}

	public void setProductIsAlwaysFreeShipping(boolean productIsAlwaysFreeShipping) {
		this.productIsAlwaysFreeShipping = productIsAlwaysFreeShipping;
	}

	public Integer getProductQuantityOrderMin() {
		return productQuantityOrderMin;
	}

	public void setProductQuantityOrderMin(Integer productQuantityOrderMin) {
		this.productQuantityOrderMin = productQuantityOrderMin;
	}

	public Integer getProductQuantityOrderMax() {
		return productQuantityOrderMax;
	}

	public void setProductQuantityOrderMax(Integer productQuantityOrderMax) {
		this.productQuantityOrderMax = productQuantityOrderMax;
	}

	@Override
	public Long getId() {
		return id;
	}

	@Override
	public void setId(Long id) {
		this.id = id;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public Set<ProductPrice> getPrices() {
		return prices;
	}

	public void setPrices(Set<ProductPrice> prices) {
		this.prices = prices;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}

	public String getOwner() {
		return owner;
	}

	public void setOwner(String owner) {
		this.owner = owner;
	}

	@Override
	public AuditSection getAuditSection() {
		return auditSection;
	}

	@Override
	public void setAuditSection(AuditSection audit) {
		this.auditSection = audit;

	}

	public Boolean getAvailable() {
		return available;
	}

	public void setAvailable(Boolean available) {
		this.available = available;
	}

	public String getSku() {
		return sku;
	}

	public void setSku(String sku) {
		this.sku = sku;
	}

	public ProductDimensions getDimensions() {
		return dimensions;
	}

	public void setDimensions(ProductDimensions dimensions) {
		this.dimensions = dimensions;
	}

	public ProductVariant getProductVariant() {
		return productVariant;
	}

	public void setProductVariant(ProductVariant productVariant) {
		this.productVariant = productVariant;
	}


}



```
