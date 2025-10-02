# Product.java

## Review

## 1. Summary
**Purpose**  
`Product` is a JPA entity that represents a catalog product in the SalesManager e‑commerce platform.  
It contains all the data needed to describe a product, its pricing, inventory, media, relationships, and taxonomy, and is persisted to the `PRODUCT` table.

**Key components**

| Component | Role |
|-----------|------|
| `@Entity` & mapping annotations | Persist the domain model to a relational database. |
| `AuditSection` | Auditing information (created/modified timestamps, user IDs). |
| Collections (`Set`) | One‑to‑many relationships with descriptions, attributes, images, availability, relationships, variants, and many‑to‑many with categories. |
| `MerchantStore` | The owning store – required for uniqueness of SKU. |
| Enums (`ProductCondition`, `RentalStatus`) | Domain specific constraints. |
| Helper getters (`getProductDescription`, `getProductImage`) | Convenience methods for retrieving the primary description / image. |

**Design patterns / libraries**

* **JPA/Hibernate** – ORM framework, uses annotations for entity mapping.  
* **Composition over inheritance** – many domain objects (Description, Attribute, etc.) are separate entities.  
* **Lazy loading** – most associations are `FetchType.LAZY` to avoid unnecessary data loads.  
* **Audit listener** – `AuditListener` automatically populates audit fields before persist/merge.  

## 2. Detailed Description
### Entity layout
The `Product` entity is stored in the `PRODUCT` table.  
The primary key `PRODUCT_ID` is generated using a table generator (`SM_SEQUENCER`).  
A unique constraint on (`MERCHANT_ID`, `SKU`) ensures that each store has distinct SKUs.

### Relationships
| Relationship | Cardinality | Cascade | Notes |
|--------------|-------------|--------|-------|
| `descriptions` | 1‑N | ALL | Each product has many language‑specific descriptions. |
| `availabilities` | 1‑N | ALL | Inventory data per location/variant. |
| `attributes` | 1‑N | ALL | Extra product properties. |
| `images` | 1‑N | REMOVE | Images are explicitly removed when a product is deleted. |
| `relationships` | 1‑N | ALL | Related products / bundles. |
| `categories` | N‑M | REFRESH | Product‑category join table `PRODUCT_CATEGORY`. |
| `merchantStore` | N‑1 | REFRESH | The owning merchant store. |
| `manufacturer`, `type`, `taxClass`, `owner` | N‑1 | REFRESH | Optional parent entities. |
| `variants` | 1‑N | ALL | Product variants (size, color, etc.). |

### Lifecycle
1. **Construction** – The no‑arg constructor is used by Hibernate.  
2. **Persist** – `AuditListener` sets created/modified timestamps.  
3. **Update** – `setAuditSection` is called automatically; cascades propagate to child collections.  
4. **Delete** – Images are removed explicitly, others follow cascade rules.

### Constraints & validation
* `sku` is mandatory and must match `^[a-zA-Z0-9_]*$`.  
* `productDescription` / `productImage` helper methods safely guard against null/empty collections.  

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `getId`, `setId` | Primary key accessor | `Long id` | `Long` | None |
| `getAuditSection`, `setAuditSection` | Auditing fields | `AuditSection` | `AuditSection` | None |
| **Getters** (`getProductLength`, `getSku`, …) | Return field values | – | field value | None |
| **Setters** (`setProductLength`, `setSku`, …) | Mutate field values | new value | void | None |
| `getProductDescription` | Convenience: returns first `ProductDescription` | – | `ProductDescription` | None |
| `getProductImage` | Convenience: returns default image or first image | – | `ProductImage` | None |
| `isProductVirtual`, `getProductVirtual` | Alias for the flag | – | boolean | None |
| `isPreOrder`, `setPreOrder` | Accessor for pre‑order flag | – | boolean / void | None |
| `isProductShipeable`, `setProductShipeable` | Accessor for shippable flag | – | boolean / void | None |
| `getOwner`, `setOwner` | Accessor for owning customer | – | `Customer` / void | None |
| `getRentalPeriod`, `setRentalPeriod` | Accessor for rental period | – | Integer / void | None |
| `getRentalDuration`, `setRentalDuration` | Accessor for rental duration | – | Integer / void | None |
| `getCondition`, `setCondition` | Accessor for product condition enum | – | `ProductCondition` / void | None |
| `getRentalStatus`, `setRentalStatus` | Accessor for rental status enum | – | `RentalStatus` / void | None |
| `setAvailable`, `isAvailable` | Accessor for availability flag | – | boolean / void | None |

### Reusable utilities
The class contains no generic helper methods; all methods are specific to the `Product` entity.

## 4. Dependencies
| Library / API | Type | Usage |
|---------------|------|-------|
| `javax.persistence` | JPA | Entity mapping and persistence. |
| `javax.validation.constraints` | Bean Validation | Validate `sku` field. |
| `org.hibernate.annotations.Cascade` | Hibernate | Fine‑grained cascade control. |
| `com.salesmanager.core.model.*` | Project internal | Domain entities: `Category`, `ProductAttribute`, `ProductImage`, etc. |
| `com.salesmanager.core.model.common.audit.*` | Internal | Auditing support. |
| `java.util.*`, `java.math.*` | Standard Java | Collections, dates, numbers. |

All dependencies are either standard JDK/JPA/Bean‑Validation or internal to the SalesManager codebase. No external, platform‑specific libraries are required.

## 5. Additional Notes & Recommendations
### Strengths
* **Clear mapping** – annotations fully describe table, column, and relationship semantics.  
* **Lazy loading** – prevents unnecessary data fetches.  
* **Auditing** – handled centrally via `AuditListener`.  
* **Validation** – SKU format is enforced.  

### Potential issues / edge cases
1. **Missing cascade for `categories`** – Only `REFRESH` is cascaded. If a category is deleted, the join row is not automatically removed, potentially leaving stale references. Consider `CascadeType.PERSIST, MERGE, REMOVE` if desired.  
2. **`ProductImage` cascade REMOVE** – Works when product is deleted, but may leave orphaned image files on disk if file deletion logic is not handled elsewhere. Ensure a cleanup service runs.  
3. **`getProductDescription`** – Assumes the first description is the default. If multiple locales exist, this may return a non‑intended description. A `locale` parameter would be safer.  
4. **`getProductImage`** – Iterates through all images each call; could be cached or optimized.  
5. **Field defaults** – Several numeric fields are `null` by default; database columns should allow `NULL` or default to zero to avoid `NullPointerException`.  
6. **Redundant getters** – `isProductVirtual` and `getProductVirtual` both expose the same flag. Consider removing one.  
7. **Boolean wrappers** – `setAvailable(Boolean)` vs `setAvailable(boolean)` – keep a single method to avoid confusion.  
8. **AuditListener** – Not shown here; ensure it correctly handles both creation and updates and respects the `AuditSection` lifecycle.  

### Future enhancements
* **Validation for numeric ranges** – Add `@Min`, `@Max` or custom validators for dimensions, weight, and price.  
* **Soft delete** – Add an `active` flag to allow deactivation without physical removal.  
* **Search indexing** – Integrate with ElasticSearch or Solr for full‑text search of product names/descriptions.  
* **Unit of work** – Consider using DTOs or projections for read‑only views to reduce payload size.  
* **Internationalization** – Store descriptions and images per locale; provide locale‑aware helpers.  

Overall, the `Product` entity is well‑structured and follows standard JPA practices. Minor cleanup of duplicate methods and cascade rules will improve maintainability and data integrity.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.catalog.product;

import java.math.BigDecimal;
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
import javax.persistence.JoinColumn;
import javax.persistence.JoinTable;
import javax.persistence.ManyToMany;
import javax.persistence.ManyToOne;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.persistence.UniqueConstraint;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.Pattern;

import org.hibernate.annotations.Cascade;

import com.salesmanager.core.model.catalog.category.Category;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.description.ProductDescription;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.catalog.product.manufacturer.Manufacturer;
import com.salesmanager.core.model.catalog.product.relationship.ProductRelationship;
import com.salesmanager.core.model.catalog.product.type.ProductType;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.common.audit.AuditListener;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.tax.taxclass.TaxClass;


@Entity
@EntityListeners(value = AuditListener.class)
@Table(name = "PRODUCT", uniqueConstraints=
@UniqueConstraint(columnNames = {"MERCHANT_ID", "SKU"}))
public class Product extends SalesManagerEntity<Long, Product> implements Auditable {
	private static final long serialVersionUID = 1L;

	@Id
	@Column(name = "PRODUCT_ID", unique=true, nullable=false)
	@TableGenerator(
		 name = "TABLE_GEN", 
		 table = "SM_SEQUENCER", 
		 pkColumnName = "SEQ_NAME", 
		 valueColumnName = "SEQ_COUNT", 
		 pkColumnValue = "PRODUCT_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;

	@Embedded
	private AuditSection auditSection = new AuditSection();

	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "product")
	private Set<ProductDescription> descriptions = new HashSet<ProductDescription>();
	
	/**
	 * Inventory
	 */
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy="product")
	private Set<ProductAvailability> availabilities = new HashSet<ProductAvailability>();

	/**
	 * Attributes of a product
	 * Decorates the product with additional properties
	 */
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "product")
	private Set<ProductAttribute> attributes = new HashSet<ProductAttribute>();
	
	/**
	 * Default product images
	 */
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.REMOVE, mappedBy = "product")//cascade is set to remove because product save requires logic to create physical image first and then save the image id in the database, cannot be done in cascade
	private Set<ProductImage> images = new HashSet<ProductImage>();

	/**
	 * Related items / product groups
	 */
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "product")
	private Set<ProductRelationship> relationships = new HashSet<ProductRelationship>();

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name="MERCHANT_ID", nullable=false)
	private MerchantStore merchantStore;
	
	/**
	 * Product to category
	 */
	@ManyToMany(fetch=FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinTable(name = "PRODUCT_CATEGORY", joinColumns = { 
			@JoinColumn(name = "PRODUCT_ID", nullable = false, updatable = false) }
			, 
			inverseJoinColumns = { @JoinColumn(name = "CATEGORY_ID", 
					nullable = false, updatable = false) }
	)
	@Cascade({
		org.hibernate.annotations.CascadeType.DETACH,
		org.hibernate.annotations.CascadeType.LOCK,
		org.hibernate.annotations.CascadeType.REFRESH,
		org.hibernate.annotations.CascadeType.REPLICATE
		
	})
	private Set<Category> categories = new HashSet<Category>();
	
	/**
	 * Product variants
	 * Decorates the product with variants
	 * 
	 */
	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "product")
	private Set<ProductVariant> variants = new HashSet<ProductVariant>();
	
	@Column(name="DATE_AVAILABLE")
	@Temporal(TemporalType.TIMESTAMP)
	private Date dateAvailable = new Date();
	
	
	@Column(name = "AVAILABLE")
	private boolean available = true;
	

	@Column(name = "PREORDER")
	private boolean preOrder = false;
	

	@ManyToOne(fetch = FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinColumn(name="MANUFACTURER_ID", nullable=true)
	private Manufacturer manufacturer;

	@ManyToOne(fetch = FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinColumn(name="PRODUCT_TYPE_ID", nullable=true)
	private ProductType type;

	@ManyToOne(fetch = FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinColumn(name="TAX_CLASS_ID", nullable=true)
	private TaxClass taxClass;

	@Column(name = "PRODUCT_VIRTUAL")
	private boolean productVirtual = false;
	
	@Column(name = "PRODUCT_SHIP")
	private boolean productShipeable = false;

	@Column(name = "PRODUCT_FREE")
	private boolean productIsFree;

	@Column(name = "PRODUCT_LENGTH")
	private BigDecimal productLength;

	@Column(name = "PRODUCT_WIDTH")
	private BigDecimal productWidth;

	@Column(name = "PRODUCT_HEIGHT")
	private BigDecimal productHeight;

	@Column(name = "PRODUCT_WEIGHT")
	private BigDecimal productWeight;

	@Column(name = "REVIEW_AVG")
	private BigDecimal productReviewAvg;

	@Column(name = "REVIEW_COUNT")
	private Integer productReviewCount;

	@Column(name = "QUANTITY_ORDERED")
	private Integer productOrdered;
	
	@Column(name = "SORT_ORDER")
	private Integer sortOrder = new Integer(0);

	@NotEmpty
	@Pattern(regexp="^[a-zA-Z0-9_]*$")
	@Column(name = "SKU")
	private String sku;
	
	/**
	 * External system reference SKU/ID
	 */
	@Column(name = "REF_SKU")
	private String refSku;
	
	@Column(name="COND", nullable = true)
	private ProductCondition condition;
	
	/**
	 * RENTAL ADDITIONAL FIELDS
	 */

	@Column(name="RENTAL_STATUS", nullable = true)
	private RentalStatus rentalStatus;
	

	@Column(name="RENTAL_DURATION", nullable = true)
	private Integer rentalDuration;
	
	@Column(name="RENTAL_PERIOD", nullable = true)
	private Integer rentalPeriod;

	
	public Integer getRentalPeriod() {
		return rentalPeriod;
	}

	public void setRentalPeriod(Integer rentalPeriod) {
		this.rentalPeriod = rentalPeriod;
	}

	public Integer getRentalDuration() {
		return rentalDuration;
	}

	public void setRentalDuration(Integer rentalDuration) {
		this.rentalDuration = rentalDuration;
	}

	/**
	 * End rental fields
	 */
	
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name="CUSTOMER_ID", nullable=true)
	private Customer owner;

	public Product() {
	}

	@Override
	public Long getId() {
		return this.id;
	}

	@Override
	public void setId(Long id) {
		this.id = id;
	}

	@Override
	public AuditSection getAuditSection() {
		return auditSection;
	}

	@Override
	public void setAuditSection(AuditSection auditSection) {
		this.auditSection = auditSection;
	}


	public boolean isProductVirtual() {
		return productVirtual;
	}



	public BigDecimal getProductLength() {
		return productLength;
	}

	public void setProductLength(BigDecimal productLength) {
		this.productLength = productLength;
	}

	public BigDecimal getProductWidth() {
		return productWidth;
	}

	public void setProductWidth(BigDecimal productWidth) {
		this.productWidth = productWidth;
	}

	public BigDecimal getProductHeight() {
		return productHeight;
	}

	public void setProductHeight(BigDecimal productHeight) {
		this.productHeight = productHeight;
	}

	public BigDecimal getProductWeight() {
		return productWeight;
	}

	public void setProductWeight(BigDecimal productWeight) {
		this.productWeight = productWeight;
	}

	public BigDecimal getProductReviewAvg() {
		return productReviewAvg;
	}

	public void setProductReviewAvg(BigDecimal productReviewAvg) {
		this.productReviewAvg = productReviewAvg;
	}

	public Integer getProductReviewCount() {
		return productReviewCount;
	}

	public void setProductReviewCount(Integer productReviewCount) {
		this.productReviewCount = productReviewCount;
	}


	public Integer getProductOrdered() {
		return productOrdered;
	}

	public void setProductOrdered(Integer productOrdered) {
		this.productOrdered = productOrdered;
	}

	public String getSku() {
		return sku;
	}

	public void setSku(String sku) {
		this.sku = sku;
	}

	public Set<ProductDescription> getDescriptions() {
		return descriptions;
	}

	public void setDescriptions(Set<ProductDescription> descriptions) {
		this.descriptions = descriptions;
	}


	public boolean getProductVirtual() {
		return productVirtual;
	}

	public void setProductVirtual(boolean productVirtual) {
		this.productVirtual = productVirtual;
	}

	public boolean getProductIsFree() {
		return productIsFree;
	}

	public void setProductIsFree(boolean productIsFree) {
		this.productIsFree = productIsFree;
	}



	public Set<ProductAttribute> getAttributes() {
		return attributes;
	}

	public void setAttributes(Set<ProductAttribute> attributes) {
		this.attributes = attributes;
	}



	public Manufacturer getManufacturer() {
		return manufacturer;
	}

	public void setManufacturer(Manufacturer manufacturer) {
		this.manufacturer = manufacturer;
	}

	public ProductType getType() {
		return type;
	}

	public void setType(ProductType type) {
		this.type = type;
	}



	public Set<ProductAvailability> getAvailabilities() {
		return availabilities;
	}

	public void setAvailabilities(Set<ProductAvailability> availabilities) {
		this.availabilities = availabilities;
	}

	public TaxClass getTaxClass() {
		return taxClass;
	}

	public void setTaxClass(TaxClass taxClass) {
		this.taxClass = taxClass;
	}

	public Set<ProductImage> getImages() {
		return images;
	}

	public void setImages(Set<ProductImage> images) {
		this.images = images;
	}

	public Set<ProductRelationship> getRelationships() {
		return relationships;
	}

	public void setRelationships(Set<ProductRelationship> relationships) {
		this.relationships = relationships;
	}


	public Set<Category> getCategories() {
		return categories;
	}

	public void setCategories(Set<Category> categories) {
		this.categories = categories;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}



	public Date getDateAvailable() {
		return dateAvailable;
	}

	public void setDateAvailable(Date dateAvailable) {
		this.dateAvailable = dateAvailable;
	}

	public void setSortOrder(Integer sortOrder) {
		this.sortOrder = sortOrder;
	}

	public Integer getSortOrder() {
		return sortOrder;
	}



	public void setAvailable(Boolean available) {
		this.available = available;
	}

	public boolean isAvailable() {
		return available;
	}
	
	public boolean isProductShipeable() {
		return productShipeable;
	}

	public void setProductShipeable(Boolean productShipeable) {
		this.productShipeable = productShipeable;
	}

	
	public ProductDescription getProductDescription() {
		if(this.getDescriptions()!=null && this.getDescriptions().size()>0) {
			return this.getDescriptions().iterator().next();
		}
		return null;
	}
	
	public ProductImage getProductImage() {
		ProductImage productImage = null;
		if(this.getImages()!=null && this.getImages().size()>0) {
			for(ProductImage image : this.getImages()) {
				productImage = image;
				if(productImage.isDefaultImage()) {
					break;
				}
			}
		}
		return productImage;
	}
	
	public boolean isPreOrder() {
		return preOrder;
	}

	public void setPreOrder(boolean preOrder) {
		this.preOrder = preOrder;
	}

	public String getRefSku() {
		return refSku;
	}

	public void setRefSku(String refSku) {
		this.refSku = refSku;
	}

	public ProductCondition getCondition() {
		return condition;
	}

	public void setCondition(ProductCondition condition) {
		this.condition = condition;
	}

	public RentalStatus getRentalStatus() {
		return rentalStatus;
	}

	public void setRentalStatus(RentalStatus rentalStatus) {
		this.rentalStatus = rentalStatus;
	}
	
	public Customer getOwner() {
		return owner;
	}

	public void setOwner(Customer owner) {
		this.owner = owner;
	}

	public Set<ProductVariant> getVariants() {
		return variants;
	}

	public void setVariants(Set<ProductVariant> variants) {
		this.variants = variants;
	}

	public void setAvailable(boolean available) {
		this.available = available;
	}

	public void setProductShipeable(boolean productShipeable) {
		this.productShipeable = productShipeable;
	}




}



```
