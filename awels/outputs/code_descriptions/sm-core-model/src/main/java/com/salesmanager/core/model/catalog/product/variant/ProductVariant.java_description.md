# ProductVariant.java

## Review

## 1. Summary

**Purpose**  
`ProductVariant` represents a single variant of a product in an e‑commerce catalogue.  
It ties a product to a specific set of product variations (size, color, etc.), carries SKU information, stock availability, and audit data.

**Key Components**

| Component | Role |
|-----------|------|
| `@Entity` + JPA annotations | Maps the class to the `PRODUCT_VARIANT` table. |
| `AuditSection` | Stores created/updated timestamps and user IDs. |
| `ProductVariation`, `ProductVariationValue`, `Product` | Relationships that link a variant to its parent product and the selected variation values. |
| `ProductAvailability` | One‑to‑many set of availability records (stock, status, etc.). |
| Validation annotations (`@NotEmpty`, `@Pattern`) | Ensure SKU is non‑empty and follows the allowed pattern. |

**Design Patterns / Frameworks**

- **JPA/Hibernate** entity mapping.
- **Auditable** interface with an `AuditListener` for automatic timestamp handling.
- **Embeddable** `AuditSection` pattern.
- Standard **Domain‑Driven Design** approach: entities with business logic and relationships.

---

## 2. Detailed Description

### Entity Mapping

- **Table**: `PRODUCT_VARIANT` with a unique constraint on `(PRODUCT_ID, SKU)` and an index on `PRODUCT_ID`.
- **Primary Key**: Generated via a table generator `PRODUCT_VAR_SEQ_NEXT_VAL` in `SM_SEQUENCER`.
- **Columns**:
  - `PRODUCT_VARIANT_ID`, `DATE_AVAILABLE`, `AVAILABLE`, `DEFAULT_SELECTION`, `CODE`, `SORT_ORDER`, `SKU`.
- **Relationships**:
  - `ProductVariation` (optional) – the variant group (e.g., “Size”).
  - `ProductVariation` `variationValue` – the chosen value (e.g., “Large”).
  - `Product` – parent product (mandatory).
  - `ProductVariantGroup` – grouping of variants (optional).
  - `Set<ProductAvailability>` – stock records (cascade all).

### Execution Flow

1. **Creation**: When a `ProductVariant` instance is persisted, `AuditListener` automatically populates the audit section.
2. **Usage**: Service layers typically fetch a product, then its variants, apply business rules (e.g., default selection), and manage availability.
3. **Update**: Modifications to fields like `sku`, `available`, or `sortOrder` trigger JPA dirty checks; audit fields are updated automatically.
4. **Deletion**: Cascade type on `availabilities` ensures that deleting a variant removes its availability records.

### Assumptions & Constraints

- `SKU` must be unique per product and non‑empty; pattern restricts to alphanumerics and underscores.
- `available` defaults to `true`, implying the variant is active unless explicitly disabled.
- `defaultSelection` indicates the variant chosen when multiple variants exist.
- Relationships use **lazy loading** except for `availabilities`, which are fetched lazily as well but may be eager in specific contexts.
- The entity extends `SalesManagerEntity`, providing an `id` field and generic persistence utilities.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getAuditSection()` | Retrieves audit data. | None | `AuditSection` | None |
| `setAuditSection(AuditSection)` | Sets audit data. | `AuditSection audit` | void | Overwrites `auditSection` |
| `getId()` | JPA primary key getter. | None | `Long` | None |
| `setId(Long)` | JPA primary key setter. | `Long id` | void | Sets `this.id` |
| `getDateAvailable()` | Getter for availability date. | None | `Date` | None |
| `setDateAvailable(Date)` | Setter for availability date. | `Date dateAvailable` | void | Sets `this.dateAvailable` |
| `isAvailable()` | Availability flag. | None | `boolean` | None |
| `setAvailable(boolean)` | Sets availability flag. | `boolean available` | void | Sets `this.available` |
| `getVariation()` | Getter for variant group. | None | `ProductVariation` | None |
| `setVariation(ProductVariation)` | Setter for variant group. | `ProductVariation variation` | void | Sets `this.variation` |
| `getProduct()` | Getter for parent product. | None | `Product` | None |
| `setProduct(Product)` | Setter for parent product. | `Product product` | void | Sets `this.product` |
| `getVariationValue()` | Getter for chosen variation value. | None | `ProductVariation` | None |
| `setVariationValue(ProductVariation)` | Setter for variation value. | `ProductVariation variationValue` | void | Sets `this.variationValue` |
| `isDefaultSelection()` | Flag indicating default variant. | None | `boolean` | None |
| `setDefaultSelection(boolean)` | Sets default flag. | `boolean defaultSelection` | void | Sets `this.defaultSelection` |
| `getSku()` | SKU getter. | None | `String` | None |
| `setSku(String)` | SKU setter. | `String sku` | void | Sets `this.sku` |
| `getSortOrder()` | Sort order getter. | None | `Integer` | None |
| `setSortOrder(Integer)` | Sort order setter. | `Integer sortOrder` | void | Sets `this.sortOrder` |
| `getCode()` | Code getter. | None | `String` | None |
| `setCode(String)` | Code setter. | `String code` | void | Sets `this.code` |
| `getProductVariantGroup()` | Getter for variant group. | None | `ProductVariantGroup` | None |
| `setProductVariantGroup(ProductVariantGroup)` | Setter for variant group. | `ProductVariantGroup productVariantGroup` | void | Sets `this.productVariantGroup` |
| `getAvailabilities()` | Returns set of availability records. | None | `Set<ProductAvailability>` | None |
| `setAvailabilities(Set<ProductAvailability>)` | Sets availability set. | `Set<ProductAvailability> availabilities` | void | Overwrites `this.availabilities` |

*All setters are simple field assignments; getters return the current value. No business logic is embedded, keeping the entity anemic.*

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.*` | JPA / Hibernate | Standard persistence API. |
| `javax.validation.constraints.*` | Bean Validation | JSR‑380, typically Hibernate Validator. |
| `com.salesmanager.core.model.*` | Project internals | Domain entities (`Product`, `ProductVariation`, etc.). |
| `com.salesmanager.core.model.common.audit.*` | Auditing utilities | `AuditListener`, `AuditSection`, `Auditable`. |
| `com.salesmanager.core.model.generic.SalesManagerEntity` | Base entity | Provides `id` handling and possible generic DAO utilities. |
| `java.util.*` | Standard library | Collections, Date. |

No external third‑party libraries beyond standard JPA/validation stack and the internal SalesManager modules.

---

## 5. Additional Notes

### Strengths

- **Clear separation** of concerns: entity mapping, validation, and auditing are distinct.
- **Consistency** with other domain entities (use of `SalesManagerEntity`).
- **Auditing** automatically handled via `AuditListener`.

### Potential Improvements / Edge Cases

| Issue | Suggested Fix |
|-------|---------------|
| **`Date` usage** | Replace `java.util.Date` with `java.time.Instant` or `LocalDateTime` to avoid legacy date pitfalls and improve timezone handling. |
| **Cascade on `availabilities`** | Consider `CascadeType.PERSIST, MERGE` only; `REMOVE` may unintentionally delete availability records when a variant is deleted (if not intended). |
| **`variation` and `variationValue` both typed `ProductVariation`** | Might be clearer to use a separate entity for variation values (e.g., `ProductVariationValue`) to avoid confusion. |
| **No `equals`/`hashCode`** | Implement these based on business keys (`product`, `sku`) to support correct set operations. |
| **Mutable `Date`** | Expose defensive copies in getters/setters to prevent external mutation of internal state. |
| **Validation on `sku`** | Consider adding a uniqueness constraint at the database level (already present) but also enforce it at the application level to provide immediate feedback. |
| **`available` flag default** | Explicitly initialize in constructor or via JPA default to avoid null issues in some providers. |
| **`code` column nullable** | Document its purpose; if it's optional, provide default logic where appropriate. |

### Future Enhancements

- **Soft delete**: Add a `deleted` flag and filter out soft‑deleted variants instead of physical removal.
- **Versioning / optimistic locking**: Add `@Version` field for concurrent updates.
- **Localized attributes**: Separate `code` and `sku` into localized entities if needed.
- **Business logic**: Introduce service‑level validation for default selection and SKU uniqueness before persistence.
- **DTO mapping**: Create lightweight DTOs for API exposure, avoiding direct entity exposure.

Overall, the class is well‑structured for a JPA domain entity, with clear mappings and auditing. Minor refactors around date handling and equality would enhance robustness.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.catalog.product.variant;

import java.util.Date;
import java.util.HashSet;
import java.util.Set;

import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.EntityListeners;
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
import javax.persistence.UniqueConstraint;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.Pattern;

import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.variation.ProductVariation;
import com.salesmanager.core.model.common.audit.AuditListener;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.generic.SalesManagerEntity;


@Entity
@EntityListeners(value = AuditListener.class)
@Table(name = "PRODUCT_VARIANT",
indexes = @Index(columnList = "PRODUCT_ID"),
uniqueConstraints = 
        @UniqueConstraint(columnNames = { 
        "PRODUCT_ID",
		"SKU" }))
public class ProductVariant extends SalesManagerEntity<Long, ProductVariant> implements Auditable {
	private static final long serialVersionUID = 1L;

	@Id
	@Column(name = "PRODUCT_VARIANT_ID", unique = true, nullable = false)
	@TableGenerator(name = "TABLE_GEN", 
	table = "SM_SEQUENCER", 
	pkColumnName = "SEQ_NAME", 
	valueColumnName = "SEQ_COUNT", 
	pkColumnValue = "PRODUCT_VAR_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;


	@Embedded
	private AuditSection auditSection = new AuditSection();

	@Column(name = "DATE_AVAILABLE")
	@Temporal(TemporalType.TIMESTAMP)
	private Date dateAvailable = new Date();
	
	@Column(name = "AVAILABLE")
	private boolean available = true;
	
	@Column(name = "DEFAULT_SELECTION")
	private boolean defaultSelection = true;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "PRODUCT_VARIATION_ID", nullable = true)
	private ProductVariation variation;

	@ManyToOne(targetEntity = Product.class)
	@JoinColumn(name = "PRODUCT_ID", nullable = false)
	private Product product;
	
	@Column(name = "CODE", nullable = true)
	private String code;
	
	@Column(name="SORT_ORDER")
	private Integer sortOrder = 0;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "PRODUCT_VARIATION_VALUE_ID", nullable = true)
	private ProductVariation variationValue;

	@NotEmpty
	@Pattern(regexp="^[a-zA-Z0-9_]*$")
	@Column(name = "SKU")
	private String sku;
	
	@ManyToOne(targetEntity = ProductVariantGroup.class)
	@JoinColumn(name = "PRODUCT_VARIANT_GROUP_ID", nullable = true)
	private ProductVariantGroup productVariantGroup;
	
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy="productVariant")
	private Set<ProductAvailability> availabilities = new HashSet<ProductAvailability>();


	@Override
	public AuditSection getAuditSection() {
		return auditSection;
	}

	@Override
	public void setAuditSection(AuditSection audit) {
		this.auditSection = audit;

	}

	@Override
	public Long getId() {
		return this.id;
	}

	@Override
	public void setId(Long id) {
		this.id = id;

	}

	public Date getDateAvailable() {
		return dateAvailable;
	}

	public void setDateAvailable(Date dateAvailable) {
		this.dateAvailable = dateAvailable;
	}

	public boolean isAvailable() {
		return available;
	}

	public void setAvailable(boolean available) {
		this.available = available;
	}

	public ProductVariation getVariation() {
		return variation;
	}

	public void setVariation(ProductVariation variation) {
		this.variation = variation;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public ProductVariation getVariationValue() {
		return variationValue;
	}

	public void setVariationValue(ProductVariation variationValue) {
		this.variationValue = variationValue;
	}

	public boolean isDefaultSelection() {
		return defaultSelection;
	}

	public void setDefaultSelection(boolean defaultSelection) {
		this.defaultSelection = defaultSelection;
	}

	public String getSku() {
		return sku;
	}

	public void setSku(String sku) {
		this.sku = sku;
	}

	public Integer getSortOrder() {
		return sortOrder;
	}

	public void setSortOrder(Integer sortOrder) {
		this.sortOrder = sortOrder;
	}


	public String getCode() { return code; }
	 
	public void setCode(String code) { this.code = code; }

	public ProductVariantGroup getProductVariantGroup() {
		return productVariantGroup;
	}

	public void setProductVariantGroup(ProductVariantGroup productVariantGroup) {
		this.productVariantGroup = productVariantGroup;
	}

	public Set<ProductAvailability> getAvailabilities() {
		return availabilities;
	}

	public void setAvailabilities(Set<ProductAvailability> availabilities) {
		this.availabilities = availabilities;
	}
	



}



```
