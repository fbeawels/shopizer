# ProductEntity.java

## Review

## 1. Summary  
The **`ProductEntity`** class is a thin, serializable DTO that augments the domain‑model **`Product`** with data‑specific fields used by the services API.  
It adds bookkeeping information such as pricing, stock, SKU, ordering constraints, and rental metadata, along with user‑facing fields such as rating and product type flags.

**Key components**  
| Component | Purpose |
|-----------|---------|
| `price`, `quantity`, `sku` | Basic e‑commerce data |
| `preOrder`, `productVirtual`, `productIsFree` | Flags to drive catalog logic |
| `quantityOrderMaximum/Minimum` | Order quantity constraints |
| `productSpecifications` | Nested product attributes (type not shown) |
| `rating`, `ratingCount` | Aggregated review data |
| `rentalDuration`, `rentalPeriod` | Rental‑specific attributes |

The class uses plain Java SE features; no frameworks or external libraries are required beyond the base JDK. It follows standard JavaBean conventions and is ready for use in serialization, caching, or ORM contexts.

---

## 2. Detailed Description  
### Class hierarchy  
`ProductEntity` extends `Product` – the base domain entity – and adds only mutable, API‑level fields. The parent class likely contains core identifiers (id, name, description, etc.) that are not visible here.

### Execution flow  
1. **Construction** – Default constructor (inherited from `Product`) creates an empty entity.  
2. **Population** – Service layer code sets each property via the public setters.  
3. **Usage** – The entity is passed to DTO adapters, serialization frameworks, or persistence layers.  
4. **Cleanup** – No explicit resource management; garbage collector handles object lifecycle.

### Assumptions & constraints  
- **Null safety** – Most fields default to primitives or non‑null objects (`rating` is initialized to `0D`). No defensive copying is performed.  
- **Validation** – No checks are performed on values (e.g., `quantity` < 0, `price` negative).  
- **Thread safety** – The class is *not* thread‑safe; concurrent mutation can cause race conditions.  
- **Serialization** – Implements `Serializable` with a static `serialVersionUID`.  
- **Extensibility** – Additional fields (e.g., promotion data) can be appended, but the current design is strictly mutable.

---

## 3. Functions/Methods  
| Method | Signature | Purpose | Inputs | Outputs | Side‑effects |
|--------|-----------|---------|--------|---------|--------------|
| `getPrice()` | `BigDecimal getPrice()` | Retrieve the product price | – | `price` | – |
| `setPrice(BigDecimal)` | `void setPrice(BigDecimal price)` | Set the product price | `price` | – | Assigns to field |
| `getQuantity()` / `setQuantity(int)` | Get/Set available stock | – | `quantity` | – | – |
| `getSku()` / `setSku(String)` | Get/Set product SKU | – | `sku` | – | – |
| `isProductIsFree()` / `setProductIsFree(boolean)` | Flag indicating free product | – | `productIsFree` | – | – |
| `getSortOrder()` / `setSortOrder(int)` | Order of product in listings | – | `sortOrder` | – | – |
| `setQuantityOrderMaximum(int)` / `getQuantityOrderMaximum()` | Max items per order | – | `quantityOrderMaximum` | – | – |
| `setProductVirtual(boolean)` / `isProductVirtual()` | Flag for virtual products | – | `productVirtual` | – | – |
| `getQuantityOrderMinimum()` / `setQuantityOrderMinimum(int)` | Min items per order | – | `quantityOrderMinimum` | – | – |
| `getRatingCount()` / `setRatingCount(int)` | Count of rating votes | – | `ratingCount` | – | – |
| `getRating()` / `setRating(Double)` | Average rating value | – | `rating` | – | – |
| `isPreOrder()` / `setPreOrder(boolean)` | Flag for pre‑order availability | – | `preOrder` | – | – |
| `getRefSku()` / `setRefSku(String)` | Reference SKU (e.g., parent SKU) | – | `refSku` | – | – |
| `getRentalDuration()` / `setRentalDuration(int)` | Rental duration in days | – | `rentalDuration` | – | – |
| `getRentalPeriod()` / `setRentalPeriod(int)` | Rental period in weeks/months | – | `rentalPeriod` | – | – |
| `getProductSpecifications()` / `setProductSpecifications(ProductSpecification)` | Nested product spec data | – | `productSpecifications` | – | – |

**Reusable/utility methods** – None; all are straightforward accessors.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `java.io.Serializable` | JDK interface | Enables serialization of the entity. |
| `java.math.BigDecimal` | JDK class | Precise monetary value. |
| `com.salesmanager.shop.model.catalog.product.Product` | Project class | Parent domain entity. |
| `com.salesmanager.shop.model.catalog.product.ProductSpecification` | Project class | Nested spec holder. |

No external libraries or frameworks are required; the class is purely JDK‑based.

---

## 5. Additional Notes  

### Strengths  
- **Simplicity** – Clear, minimal API for service consumption.  
- **Extensible** – Fields can be added without breaking existing code.  
- **Serialization‑ready** – `serialVersionUID` defined for backward compatibility.

### Areas for Improvement  

| Issue | Recommendation |
|-------|----------------|
| **Naming consistency** | Rename `productIsFree` to `isFree` or `freeProduct` and adjust the getter to `isFree()` for better readability. |
| **Null‑safety** | For reference types (`price`, `sku`, `rating`, `productSpecifications`), consider initializing with safe defaults or using `Optional`/`Objects.requireNonNull` in setters. |
| **Validation** | Enforce business rules (e.g., non‑negative price, `quantityOrderMaximum` ≥ `quantityOrderMinimum`) either in setters or via a separate validator. |
| **Immutability** | For thread‑safety and easier debugging, consider making the entity immutable (final fields, constructor‑only initialization) or using a builder pattern. |
| **`equals` / `hashCode` / `toString`** | Implement these methods to support collections, logging, and debugging. |
| **Documentation** | Add JavaDoc to each getter/setter, explaining semantics (e.g., what “pre‑order” implies). |
| **Error handling** | Throw descriptive exceptions for invalid values (e.g., `IllegalArgumentException` when setting negative quantity). |
| **Unit tests** | Provide tests covering edge cases: negative price, zero quantity, null SKU, etc. |
| **Internationalization** | If used across locales, consider `BigDecimal` scale/precision or a dedicated money type (`javax.money.Money`). |
| **Performance** | For large lists of `ProductEntity`, consider lazy loading of `productSpecifications` or caching frequently accessed fields. |

### Potential Enhancements  
- **Builder Pattern** – Simplify object creation: `ProductEntity.builder().price(...).sku(...).build();`  
- **Validation Framework** – Integrate with Bean Validation (`javax.validation`) to declaratively enforce constraints.  
- **JSON Serialization Annotations** – Add Jackson/Gson annotations if this DTO is exposed via REST APIs.  
- **Conversion Utilities** – Provide static methods to map between `ProductEntity` and domain `Product`.  

Overall, the class serves its purpose as a simple data holder, but adding defensive coding practices, documentation, and a few design‑level improvements would increase its robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.model.catalog.product.product;

import java.io.Serializable;
import java.math.BigDecimal;

import com.salesmanager.shop.model.catalog.product.Product;

/**
 * A product entity is used by services API to populate or retrieve a Product
 * entity
 * 
 * @author Carl Samson
 *
 */
public class ProductEntity extends Product implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	private BigDecimal price;
	private int quantity = 0;
	private String sku;
	private boolean preOrder = false;
	private boolean productVirtual = false;
	private int quantityOrderMaximum = -1;// default unlimited
	private int quantityOrderMinimum = 1;// default 1
	private boolean productIsFree;

	private ProductSpecification productSpecifications;
	private Double rating = 0D;
	private int ratingCount;
	private int sortOrder;
	private String refSku;


	/**
	 * RENTAL additional fields
	 * 
	 * @return
	 */

	private int rentalDuration;
	private int rentalPeriod;

	/**
	 * End RENTAL fields
	 * 
	 * @return
	 */

	public BigDecimal getPrice() {
		return price;
	}

	public void setPrice(BigDecimal price) {
		this.price = price;
	}

	public int getQuantity() {
		return quantity;
	}

	public void setQuantity(int quantity) {
		this.quantity = quantity;
	}

	public String getSku() {
		return sku;
	}

	public void setSku(String sku) {
		this.sku = sku;
	}


	public boolean isProductIsFree() {
		return productIsFree;
	}

	public void setProductIsFree(boolean productIsFree) {
		this.productIsFree = productIsFree;
	}

	public int getSortOrder() {
		return sortOrder;
	}

	public void setSortOrder(int sortOrder) {
		this.sortOrder = sortOrder;
	}

	public void setQuantityOrderMaximum(int quantityOrderMaximum) {
		this.quantityOrderMaximum = quantityOrderMaximum;
	}

	public int getQuantityOrderMaximum() {
		return quantityOrderMaximum;
	}

	public void setProductVirtual(boolean productVirtual) {
		this.productVirtual = productVirtual;
	}

	public boolean isProductVirtual() {
		return productVirtual;
	}

	public int getQuantityOrderMinimum() {
		return quantityOrderMinimum;
	}

	public void setQuantityOrderMinimum(int quantityOrderMinimum) {
		this.quantityOrderMinimum = quantityOrderMinimum;
	}

	public int getRatingCount() {
		return ratingCount;
	}

	public void setRatingCount(int ratingCount) {
		this.ratingCount = ratingCount;
	}

	public Double getRating() {
		return rating;
	}

	public void setRating(Double rating) {
		this.rating = rating;
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

	public int getRentalDuration() {
		return rentalDuration;
	}

	public void setRentalDuration(int rentalDuration) {
		this.rentalDuration = rentalDuration;
	}

	public int getRentalPeriod() {
		return rentalPeriod;
	}

	public void setRentalPeriod(int rentalPeriod) {
		this.rentalPeriod = rentalPeriod;
	}

	public ProductSpecification getProductSpecifications() {
		return productSpecifications;
	}

	public void setProductSpecifications(ProductSpecification productSpecifications) {
		this.productSpecifications = productSpecifications;
	}



}



```
