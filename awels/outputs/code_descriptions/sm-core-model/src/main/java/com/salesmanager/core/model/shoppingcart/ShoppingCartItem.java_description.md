# ShoppingCartItem.java

## Review

## 1. Summary
The `ShoppingCartItem` entity models a single line item in a customer’s shopping cart.  
* **Purpose** – Stores all data needed for cart calculations (quantity, pricing, attributes) and links back to the parent `ShoppingCart`.  
* **Key components** –  
  * **Fields** – ID, `shoppingCart`, `quantity`, `productId`/`sku`, `variant`, pricing fields (`itemPrice`, `subTotal`, `finalPrice`), attribute set, audit information.  
  * **JPA annotations** – Entity mapping, table generation, relationships (`@ManyToOne`, `@OneToMany`).  
  * **Audit support** – Implements `Auditable`, uses `AuditListener` & `AuditSection`.  
  * **Utility methods** – Add/remove attributes, setters/getters for derived price fields.  
* **Design patterns / frameworks** –  
  * JPA/Hibernate for persistence.  
  * Value‑object (`FinalPrice`) for encapsulating raw pricing data.  
  * Builder‑like constructors to ensure required state.

---

## 2. Detailed Description
### Core Components
| Component | Role |
|-----------|------|
| `@Entity` & `@Table` | Maps to `SHOPPING_CART_ITEM` table. |
| `@TableGenerator` & `@GeneratedValue` | Generates primary key via a table‑based sequence. |
| `shoppingCart` | `@ManyToOne` link back to the cart; essential for cascading operations. |
| `attributes` | `@OneToMany` set of `ShoppingCartAttributeItem` objects representing product customisations. |
| `auditSection` | Stores creation/update timestamps, user info. |
| `productId`, `sku`, `variant` | Product reference fields (legacy `productId` still present for backward‑compatibility). |
| `itemPrice`, `subTotal`, `finalPrice` | Transient fields calculated at runtime (not persisted). |
| `product`, `productVirtual`, `obsolete` | Helper fields to avoid repeated lookups; used by UI/services. |

### Flow of Execution
1. **Construction**  
   * Two public constructors: one takes `ShoppingCart` & `Product`, the other just `Product`.  
   * They set core fields (`product`, `productId`, `sku`, `quantity`, `productVirtual`).  
   * The default no‑arg constructor is deprecated to prevent incomplete instantiation.

2. **Persistence**  
   * When a `ShoppingCartItem` is saved, JPA persists all columns except transient fields.  
   * `@ManyToOne` with `fetch = EAGER` (default) links to `ShoppingCart`; cascade not specified, so parent cart must manage the relationship.  
   * `attributes` are persisted lazily; `CascadeType.ALL` ensures attribute lifecycle is tied to the item.

3. **Runtime Calculations**  
   * Pricing fields are not persisted; services compute `itemPrice`, `subTotal`, and wrap raw data into a `FinalPrice`.  
   * Methods like `addAttributes`/`removeAttributes` modify the attribute set; removal from the set triggers JPA orphan removal implicitly due to `CascadeType.ALL`.

4. **Cleanup**  
   * No explicit cleanup needed; JPA handles orphan removal and cascade deletes.

### Assumptions & Constraints
* **Backward compatibility** – The legacy `productId` column remains; new code should use `sku`.  
* **Transient fields** – Calculated at runtime; must be set by service layer before use.  
* **Concurrency** – Uses table‑based ID generation; fine for small to medium loads but can become a bottleneck under high contention.  
* **Audit** – Requires an `AuditListener` to populate timestamps.

---

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getAuditSection()/setAuditSection(AuditSection)` | Auditing support | none / `AuditSection` | `AuditSection` | None |
| `getId()/setId(Long)` | Primary key access | none / `Long` | `Long` | None |
| `setAttributes(Set<ShoppingCartAttributeItem>) / getAttributes()` | Manage custom attributes | `Set` | `Set` | Sets internal field |
| `addAttributes(ShoppingCartAttributeItem)` | Append an attribute | item | void | Adds to set |
| `removeAttributes(ShoppingCartAttributeItem)` | Remove a specific attribute | item | void | Removes from set |
| `removeAllAttributes()` | Clear attribute set | none | void | Clears set |
| `setItemPrice(BigDecimal)/getItemPrice()` | Set/get computed item price | `BigDecimal` | `BigDecimal` | Sets internal field |
| `setQuantity(Integer)/getQuantity()` | Set/get quantity | `Integer` | `Integer` | Sets internal field |
| `setShoppingCart(ShoppingCart)/getShoppingCart()` | Set/get owning cart | `ShoppingCart` | `ShoppingCart` | Sets internal field |
| `setProductId(Long)/getProductId()` | Legacy product ID | `Long` | `Long` | Sets internal field |
| `setProduct(Product)/getProduct()` | Set/get domain product | `Product` | `Product` | Sets internal field |
| `setSubTotal(BigDecimal)/getSubTotal()` | Set/get subtotal | `BigDecimal` | `BigDecimal` | Sets internal field |
| `setFinalPrice(FinalPrice)/getFinalPrice()` | Set/get detailed price | `FinalPrice` | `FinalPrice` | Sets internal field |
| `isObsolete()/setObsolete(boolean)` | Flag obsolete items | none / `boolean` | `boolean` | Sets internal flag |
| `isProductVirtual()/setProductVirtual(boolean)` | Flag if product is virtual | none / `boolean` | `boolean` | Sets internal flag |
| `getSku()/setSku(String)` | SKU accessor | none / `String` | `String` | Sets internal field |
| `getVariant()/setVariant(Long)` | Variant ID accessor | none / `Long` | `Long` | Sets internal field |
| `ShoppingCartItem(ShoppingCart, Product)` / `ShoppingCartItem(Product)` | Constructors | as per signature | new instance | Initializes required fields |
| `ShoppingCartItem()` | Deprecated default constructor | none | new instance | Leaves object partially initialised |

*Reusable utilities*: `addAttributes`, `removeAttributes`, `removeAllAttributes` can be used by any code managing attribute sets.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.*` | JPA / Hibernate | Core persistence mapping. |
| `com.fasterxml.jackson.annotation.JsonIgnore` | Jackson | Controls JSON serialization. |
| `com.salesmanager.core.model.catalog.product.*` | Domain model | `Product`, `FinalPrice`. |
| `com.salesmanager.core.model.common.audit.*` | Auditing | `AuditListener`, `AuditSection`, `Auditable`. |
| `com.salesmanager.core.model.generic.SalesManagerEntity` | Base entity | Provides common fields (e.g., `id`?). |
| `java.util.*` | Standard | Collections, BigDecimal. |

All are either standard Java or part of the same `com.salesmanager` codebase; no external third‑party libraries beyond JPA and Jackson.

---

## 5. Additional Notes
### Strengths
* **Clear separation of concerns** – Persistence fields vs. runtime calculation fields.  
* **Audit integration** – Automatic timestamp handling via `AuditListener`.  
* **Backward compatibility** – `productId` column retained; explicit deprecation signals upcoming removal.  
* **Utility methods** – Easy manipulation of attribute set.

### Potential Issues / Edge Cases
1. **Transient field consistency** – The service layer must ensure that `itemPrice`, `subTotal`, and `finalPrice` are correctly populated before any business logic uses them; otherwise, `null` values may cause `NullPointerException`.  
2. **Quantity validation** – No guard against negative or zero quantity; could lead to inconsistent cart state.  
3. **Legacy `productId`** – Keeping both `productId` and `sku` may cause confusion; code should enforce a single source of truth.  
4. **Concurrent updates** – Table‑based ID generator can be a bottleneck under high write concurrency.  
5. **Attribute orphan handling** – `CascadeType.ALL` is used, but `orphanRemoval` is not specified; ensure that removal from the set is intentional.  
6. **Obsolete flag** – No business logic shown for handling obsolete items; should be documented.

### Future Enhancements
* **Validation annotations** – Add `@NotNull`, `@Min(1)` for `quantity`.  
* **Enum for product type** – Replace boolean `productVirtual` with an enum to capture more states.  
* **Builder pattern** – Provide a fluent builder for constructing items with optional attributes.  
* **Price calculation service** – Encapsulate price logic in a dedicated service to avoid scattering calculations.  
* **Remove legacy field** – Once migration is complete, drop `productId` column and related code.  
* **Audit section integration** – Consider persisting audit data directly in the entity rather than using a separate listener.  
* **Unit tests** – Add tests for constructors, attribute manipulation, and price calculations.

Overall, the class is well‑structured for its domain purpose but would benefit from stricter validation, clearer API evolution, and improved concurrency handling.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.shoppingcart;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.Collections;
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
import javax.persistence.ManyToOne;
import javax.persistence.OneToMany;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Transient;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.common.audit.AuditListener;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.generic.SalesManagerEntity;


@Entity
@EntityListeners(value = AuditListener.class)
@Table(name = "SHOPPING_CART_ITEM")
public class ShoppingCartItem extends SalesManagerEntity<Long, ShoppingCartItem> implements Auditable, Serializable {


	private static final long serialVersionUID = 1L;

	@Id
	@Column(name = "SHP_CART_ITEM_ID", unique=true, nullable=false)
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT", pkColumnValue = "SHP_CRT_ITM_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;

	@JsonIgnore
	@ManyToOne(targetEntity = ShoppingCart.class)
	@JoinColumn(name = "SHP_CART_ID", nullable = false)
	private ShoppingCart shoppingCart;

	@Column(name="QUANTITY")
	private Integer quantity = 1;

	@Embedded
	private AuditSection auditSection = new AuditSection();

	@Deprecated //Use sku
	@Column(name="PRODUCT_ID", nullable=false) 
    private Long productId;
	
	//SKU
	@Column(name="SKU", nullable=true) 
	private String sku;

	@JsonIgnore
	@Transient
	private boolean productVirtual;

	@OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "shoppingCartItem")
	private Set<ShoppingCartAttributeItem> attributes = new HashSet<ShoppingCartAttributeItem>();
	
	@Column(name="PRODUCT_VARIANT", nullable=true)
	private Long variant;

	@JsonIgnore
	@Transient
	private BigDecimal itemPrice;//item final price including all rebates

	@JsonIgnore
	@Transient
	private BigDecimal subTotal;//item final price * quantity

	@JsonIgnore
	@Transient
	private FinalPrice finalPrice;//contains price details (raw prices)

	@JsonIgnore
	@Transient
	private Product product;

	@JsonIgnore
	@Transient
	private boolean obsolete = false;




	public ShoppingCartItem(ShoppingCart shoppingCart, Product product) {
		this(product);
		this.shoppingCart = shoppingCart;
	}

	public ShoppingCartItem(Product product) {
		this.product = product;
		this.productId = product.getId();
		this.setSku(product.getSku());
		this.quantity = 1;
		this.productVirtual = product.isProductVirtual();
	}

	/** remove usage to limit possibility to implement bugs, would use constructors above to make sure all needed attributes are set correctly **/
	@Deprecated
	public ShoppingCartItem() {
	}

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
		return id;
	}

	@Override
	public void setId(Long id) {
		this.id = id;

	}

	public void setAttributes(Set<ShoppingCartAttributeItem> attributes) {
		this.attributes = attributes;
	}

	public Set<ShoppingCartAttributeItem> getAttributes() {
		return attributes;
	}

	public void setItemPrice(BigDecimal itemPrice) {
		this.itemPrice = itemPrice;
	}

	public BigDecimal getItemPrice() {
		return itemPrice;
	}

	public void setQuantity(Integer quantity) {
		this.quantity = quantity;
	}

	public Integer getQuantity() {
		return quantity;
	}

	public ShoppingCart getShoppingCart() {
		return shoppingCart;
	}

	public void setShoppingCart(ShoppingCart shoppingCart) {
		this.shoppingCart = shoppingCart;
	}

	public void setProductId(Long productId) {
		this.productId = productId;
	}

	public Long getProductId() {
		return productId;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public Product getProduct() {
		return product;
	}

	public void addAttributes(ShoppingCartAttributeItem shoppingCartAttributeItem)
	{
	    this.attributes.add(shoppingCartAttributeItem);
	}

	public void removeAttributes(ShoppingCartAttributeItem shoppingCartAttributeItem)
	{
	    this.attributes.remove(shoppingCartAttributeItem);
	}

	public void removeAllAttributes(){
		this.attributes.removeAll(Collections.EMPTY_SET);
	}

	public void setSubTotal(BigDecimal subTotal) {
		this.subTotal = subTotal;
	}

	public BigDecimal getSubTotal() {
		return subTotal;
	}

	public void setFinalPrice(FinalPrice finalPrice) {
		this.finalPrice = finalPrice;
	}

	public FinalPrice getFinalPrice() {
		return finalPrice;
	}

	public boolean isObsolete() {
		return obsolete;
	}

	public void setObsolete(boolean obsolete) {
		this.obsolete = obsolete;
	}


	public boolean isProductVirtual() {
		return productVirtual;
	}

	public void setProductVirtual(boolean productVirtual) {
		this.productVirtual = productVirtual;
	}

	public String getSku() {
		return sku;
	}

	public void setSku(String sku) {
		this.sku = sku;
	}

	public Long getVariant() {
		return variant;
	}

	public void setVariant(Long variant) {
		this.variant = variant;
	}

}



```
