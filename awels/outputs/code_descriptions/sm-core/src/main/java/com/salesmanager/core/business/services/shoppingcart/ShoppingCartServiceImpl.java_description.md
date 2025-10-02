# ShoppingCartServiceImpl.java

## Review

## 1. Summary  
**Purpose** – `ShoppingCartServiceImpl` is the core business layer for handling customer shopping carts in a multi‑tenant e‑commerce platform.  
**Key responsibilities**

| Component | Role |
|-----------|------|
| `ShoppingCartRepository` | CRUD on `ShoppingCart` entities |
| `ShoppingCartItemRepository` | CRUD on `ShoppingCartItem` entities |
| `ShoppingCartAttributeRepository` | CRUD on `ShoppingCartAttributeItem` entities |
| `ProductService` | Load products by SKU or ID |
| `PricingService` | Calculate final prices (product + attributes) |
| `ProductAttributeService` | Load product attributes by ID |

**Design patterns / frameworks**  

* **Spring Service / Repository** – the class is annotated with `@Service` and relies on Spring Data JPA repositories.  
* **Transactional boundaries** – most business methods are annotated with `@Transactional` (read‑write or read‑only where appropriate).  
* **DTO‑ish pattern** – `ShoppingCartItem` is used as a lightweight DTO that is enriched by the service.  
* **Lazy loading & validation** – uses `Validate` from Apache Commons Lang to guard against null inputs.

---

## 2. Detailed Description  

### Flow of execution

1. **Cart retrieval**  
   * `getShoppingCart(Customer, MerchantStore)` → fetches all carts for the customer, picks the first “un‑ordered” one, populates it (`getPopulatedShoppingCart`), and removes it if it’s marked obsolete.  
   * `getById(Long, MerchantStore)` and `getByCode(String, MerchantStore)` work similarly but locate the cart by primary key or code.  

2. **Population logic** (`getPopulatedShoppingCart`)  
   * Iterates over each `ShoppingCartItem`.  
   * Calls `getPopulatedItem` to load the latest product data, update price, and clean up any stale attribute references.  
   * Flags the cart as obsolete if any item becomes obsolete.  

3. **Saving / Updating**  
   * `saveOrUpdate(ShoppingCart)` delegates to the base CRUD implementation after enriching the cart with the current IP address.  

4. **Merge** (`mergeShoppingCarts`)  
   * Merges a session‑based cart (anonymous visitor) into a logged‑in user cart.  
   * Handles quantity aggregation for duplicate products (though the current implementation has a bug described below).  

5. **Deletion**  
   * `deleteCart(ShoppingCart)` and `deleteShoppingCartItem(Long)` remove carts or single items, including their attributes.  

6. **Shipping helper** (`createShippingProduct`)  
   * Builds a list of `ShippingProduct` objects for all shippable, non‑virtual items in the cart.  

### Assumptions & constraints  

* **Single customer per cart** – a cart is expected to belong to one customer at a time.  
* **Merchant isolation** – all queries include the merchant store ID to enforce data segregation.  
* **Obsolete flag** – a cart is considered obsolete if it has no line items or any line item has been marked obsolete.  
* **Price recalculation** – every time the cart is read or modified, prices are recalculated to reflect current product data and stock levels.  

### Architecture / Design choices  

* **Separation of concerns** – database access is encapsulated in repositories; business rules live in the service.  
* **Transactional safety** – write operations are wrapped in transactions to guarantee atomicity.  
* **Use of JPA entities as DTOs** – `ShoppingCartItem` and related entities are directly manipulated and persisted, simplifying the data flow.  

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getShoppingCart(Customer, MerchantStore)` | Retrieve the active cart for a customer. | `Customer`, `MerchantStore` | `ShoppingCart` or `null` | May delete obsolete cart |
| `saveOrUpdate(ShoppingCart)` | Persist a new or existing cart. | `ShoppingCart` | – | Sets IP address, creates or updates record |
| `getById(Long, MerchantStore)` | Load cart by ID. | `Long id`, `MerchantStore` | `ShoppingCart` or `null` | May delete obsolete cart |
| `getByCode(String, MerchantStore)` | Load cart by code. | `String code`, `MerchantStore` | `ShoppingCart` or `null` | May delete obsolete cart |
| `deleteCart(ShoppingCart)` | Delete a cart by its ID. | `ShoppingCart` | – | Deletes record |
| `deleteShoppingCartItem(Long)` | Delete a cart item and its attributes. | `Long id` | – | Removes item + attributes |
| `getPopulatedShoppingCart(ShoppingCart, MerchantStore)` | Enrich cart items, recalculate prices, flag obsolete. | `ShoppingCart`, `MerchantStore` | `ShoppingCart` | May set obsolete flag, update record |
| `populateShoppingCartItem(Product, MerchantStore)` | Create a new `ShoppingCartItem` from a product. | `Product`, `MerchantStore` | `ShoppingCartItem` | – |
| `getPopulatedItem(ShoppingCartItem, MerchantStore)` | Refresh item details (product, price, attributes). | `ShoppingCartItem`, `MerchantStore` | – | May set item obsolete |
| `createShippingProduct(ShoppingCart)` | Build a list of shipping‑eligible products. | `ShoppingCart` | `List<ShippingProduct>` | – |
| `removeShoppingCart(ShoppingCart)` | Convenience wrapper for repository delete. | `ShoppingCart` | – | Deletes cart |
| `mergeShoppingCarts(ShoppingCart, ShoppingCart, MerchantStore)` | Merge a session cart into a logged‑in user cart. | `userShoppingModel`, `sessionCart`, `MerchantStore` | `ShoppingCart` | Persists merged cart, deletes session cart |
| `getShoppingCartItems(ShoppingCart, MerchantStore, ShoppingCart)` | Helper for `mergeShoppingCarts`; constructs a set of items for a cart. | `sessionCart`, `MerchantStore`, `cartModel` | `Set<ShoppingCartItem>` | – |

### Reusable / utility methods  

* `validate` from Apache Commons Lang protects against null arguments.  
* `CollectionUtils.isEmpty / isNotEmpty` guard collection operations.  
* `calculateProductPrice` from `PricingService` is the single source of truth for final pricing.  

---

## 4. Dependencies  

| External / Third‑party | Purpose | Standard/3rd‑party |
|------------------------|---------|---------------------|
| **Spring Framework** (Core, Data JPA, Transaction) | Dependency injection, transaction management, repository abstraction | 3rd‑party |
| **Hibernate / JPA** | ORM mapping for entities | 3rd‑party |
| **Apache Commons Lang / Collections** | Null checks, collection utilities | 3rd‑party |
| **SLF4J** | Logging abstraction | 3rd‑party |
| **javax.inject.Inject** | CDI style injection | Standard (JSR‑330) |
| **javax.persistence.NoResultException** | Exception handling for queries | JPA |
| **java.math.BigDecimal** | Precise monetary calculations | Standard |

No platform‑specific APIs are used beyond Spring/JPA; the code is portable across Java SE 8+ environments.

---

## 5. Additional Notes & Recommendations  

### 5.1 Potential bugs / edge‑cases  

| Issue | Impact | Suggested fix |
|-------|--------|---------------|
| **`mergeShoppingCarts` duplicate logic** – `duplicateFound` flag is reused across outer iterations, causing later unique items to be skipped. | Session cart items may not all be merged. | Reset `duplicateFound` inside the outer loop or move the duplicate check into a dedicated helper that returns a boolean. |
| **`deleteCart` overload** – calls `getById(shoppingCart.getId())` without a store argument; likely refers to a missing overloaded method. | Compile‑time error or incorrect cart lookup. | Use `getById(shoppingCart.getId(), shoppingCart.getMerchantStore())` or remove the method entirely. |
| **`getPopulatedItem`** – if `product.isProductVirtual()` is true, the item is marked as virtual but other logic (e.g., price recalculation) still runs. | Virtual products may still be processed as normal items, potentially leading to incorrect shipping or stock handling. | Short‑circuit when virtual or add guard in calling logic. |
| **Obsolete cart handling** – `getPopulatedShoppingCart` marks the cart obsolete if any item is obsolete but still returns the cart. | The calling code may continue to use a cart flagged obsolete. | Return `null` or throw an exception when the cart becomes obsolete. |
| **`findOne` usage** – deprecated in newer Spring Data JPA. | Potential deprecation warnings and future breakage. | Replace with `findById` and handle `Optional`. |
| **Null‑safety** – `shoppingCart.getLineItems()` may be `null` in some cases. | `NullPointerException`. | Enforce non‑null collections in entity constructors or guard with `Objects.requireNonNull`. |

### 5.2 Code quality & style  

| Recommendation | Reason |
|----------------|--------|
| **Use `@Transactional(readOnly = true)` on read‑only methods** (`getShoppingCart`, `getById`, `getByCode`, `getPopulatedShoppingCart`) | Improves performance and signals intent. |
| **Replace `CollectionUtils.isEmpty` with `CollectionUtils.isNotEmpty`** in the population loops for clarity. | Avoids double negation. |
| **Avoid creating new `BigDecimal` instances in tight loops** – reuse constants (`BigDecimal.ONE`, `BigDecimal.ZERO`). | Minor performance improvement. |
| **Use `Optional` for repository return types** | Reduces null‑checking boilerplate and clarifies “maybe present” semantics. |
| **Extract price calculation into a separate helper** that also sets subtotal. | Centralises monetary logic and improves testability. |
| **Add unit tests for merge logic** | Ensures correct handling of duplicates and quantity aggregation. |

### 5.3 Future enhancements  

1. **Optimistic locking** – add a version field to `ShoppingCart` to guard against concurrent updates.  
2. **Cache pricing results** – expensive price calculations could be cached per product‑attribute combination.  
3. **Event‑driven updates** – publish events when a cart or item is modified (e.g., for analytics or inventory sync).  
4. **Internationalisation support** – price calculation could respect locale‑specific currency formatting.  
5. **API layer** – expose this service through a REST controller with proper DTOs to decouple persistence from the API.  

---  

**Conclusion** – The implementation covers the core business needs of cart management, with a solid repository‑service separation and adequate transactional handling. However, several logical bugs and code‑style issues should be addressed to improve reliability, maintainability, and performance. Fixing the merge logic, cleaning up overloaded methods, and modernising repository usage will bring the codebase up to production‑ready standards.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.shoppingcart;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.shoppingcart.ShoppingCartAttributeRepository;
import com.salesmanager.core.business.repositories.shoppingcart.ShoppingCartItemRepository;
import com.salesmanager.core.business.repositories.shoppingcart.ShoppingCartRepository;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.catalog.product.attribute.ProductAttributeService;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.common.UserContext;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.shipping.ShippingProduct;
import com.salesmanager.core.model.shoppingcart.ShoppingCart;
import com.salesmanager.core.model.shoppingcart.ShoppingCartAttributeItem;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.inject.Inject;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

@Service("shoppingCartService")
public class ShoppingCartServiceImpl extends SalesManagerEntityServiceImpl<Long, ShoppingCart>
		implements ShoppingCartService {

	private ShoppingCartRepository shoppingCartRepository;

	@Inject
	private ProductService productService;

	@Inject
	private ShoppingCartItemRepository shoppingCartItemRepository;

	@Inject
	private ShoppingCartAttributeRepository shoppingCartAttributeItemRepository;

	@Inject
	private PricingService pricingService;

	@Inject
	private ProductAttributeService productAttributeService;

	private static final Logger LOGGER = LoggerFactory.getLogger(ShoppingCartServiceImpl.class);

	@Inject
	public ShoppingCartServiceImpl(ShoppingCartRepository shoppingCartRepository) {
		super(shoppingCartRepository);
		this.shoppingCartRepository = shoppingCartRepository;

	}

	/**
	 * Retrieve a {@link ShoppingCart} cart for a given customer
	 */
	@Override
	@Transactional
	public ShoppingCart getShoppingCart(final Customer customer, MerchantStore store) throws ServiceException {

		try {

			List<ShoppingCart> shoppingCarts = shoppingCartRepository.findByCustomer(customer.getId());

			// elect valid shopping cart
			List<ShoppingCart> validCart = shoppingCarts.stream().filter((cart) -> cart.getOrderId() == null)
					.collect(Collectors.toList());

			ShoppingCart shoppingCart = null;

			if (!CollectionUtils.isEmpty(validCart)) {
				shoppingCart = validCart.get(0);
				getPopulatedShoppingCart(shoppingCart, store);
				if (shoppingCart != null && shoppingCart.isObsolete()) {
					delete(shoppingCart);
					shoppingCart = null;
				}
			}

			return shoppingCart;

		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	/**
	 * Save or update a {@link ShoppingCart} for a given customer
	 */
	@Override
	public void saveOrUpdate(ShoppingCart shoppingCart) throws ServiceException {

		Validate.notNull(shoppingCart, "ShoppingCart must not be null");
		Validate.notNull(shoppingCart.getMerchantStore(), "ShoppingCart.merchantStore must not be null");

		try {
			UserContext userContext = UserContext.getCurrentInstance();
			if (userContext != null) {
				shoppingCart.setIpAddress(userContext.getIpAddress());
			}
		} catch (Exception s) {
			LOGGER.error("Cannot add ip address to shopping cart ", s);
		}

		if (shoppingCart.getId() == null || shoppingCart.getId() == 0) {
			super.create(shoppingCart);
		} else {
			super.update(shoppingCart);
		}

	}

	/**
	 * Get a {@link ShoppingCart} for a given id and MerchantStore. Will update the
	 * shopping cart prices and items based on the actual inventory. This method
	 * will remove the shopping cart if no items are attached.
	 */
	@Override
	@Transactional
	public ShoppingCart getById(final Long id, final MerchantStore store) throws ServiceException {

		try {
			ShoppingCart shoppingCart = shoppingCartRepository.findById(store.getId(), id);
			if (shoppingCart == null) {
				return null;
			}
			getPopulatedShoppingCart(shoppingCart, store);

			if (shoppingCart.isObsolete()) {
				delete(shoppingCart);
				return null;
			} else {
				return shoppingCart;
			}

		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	/**
	 * Get a {@link ShoppingCart} for a given id. Will update the shopping cart
	 * prices and items based on the actual inventory. This method will remove the
	 * shopping cart if no items are attached.
	 */
	/*
	 * @Override
	 * 
	 * @Transactional public ShoppingCart getById(final Long id, MerchantStore
	 * store) throws {
	 * 
	 * try { ShoppingCart shoppingCart = shoppingCartRepository.findOne(id); if
	 * (shoppingCart == null) { return null; }
	 * getPopulatedShoppingCart(shoppingCart);
	 * 
	 * if (shoppingCart.isObsolete()) { delete(shoppingCart); return null; } else {
	 * return shoppingCart; } } catch (Exception e) { // TODO Auto-generated catch
	 * block e.printStackTrace(); } return null;
	 * 
	 * }
	 */

	/**
	 * Get a {@link ShoppingCart} for a given code. Will update the shopping cart
	 * prices and items based on the actual inventory. This method will remove the
	 * shopping cart if no items are attached.
	 */
	@Override
	@Transactional
	public ShoppingCart getByCode(final String code, final MerchantStore store) throws ServiceException {

		try {
			ShoppingCart shoppingCart = shoppingCartRepository.findByCode(store.getId(), code);
			if (shoppingCart == null) {
				return null;
			}
			getPopulatedShoppingCart(shoppingCart, store);

			if (shoppingCart.isObsolete()) {
				delete(shoppingCart);
				return null;
			} else {
				return shoppingCart;
			}

		} catch (javax.persistence.NoResultException nre) {
			return null;
		} catch (Throwable e) {
			throw new ServiceException(e);
		}

	}

	@Override
	@Transactional
	public void deleteCart(final ShoppingCart shoppingCart) throws ServiceException {
		ShoppingCart cart = this.getById(shoppingCart.getId());
		if (cart != null) {
			super.delete(cart);
		}
	}

	/*
	 * @Override
	 * 
	 * @Transactional public ShoppingCart getByCustomer(final Customer customer)
	 * throws ServiceException {
	 * 
	 * try { List<ShoppingCart> shoppingCart =
	 * shoppingCartRepository.findByCustomer(customer.getId()); if (shoppingCart ==
	 * null) { return null; } return getPopulatedShoppingCart(shoppingCart);
	 * 
	 * } catch (Exception e) { throw new ServiceException(e); } }
	 */

	@Transactional(noRollbackFor = { org.springframework.dao.EmptyResultDataAccessException.class })
	private ShoppingCart getPopulatedShoppingCart(final ShoppingCart shoppingCart, MerchantStore store) throws Exception {

		try {

			boolean cartIsObsolete = false;
			if (shoppingCart != null) {

				Set<ShoppingCartItem> items = shoppingCart.getLineItems();
				if (items == null || items.size() == 0) {
					shoppingCart.setObsolete(true);
					return shoppingCart;

				}

				// Set<ShoppingCartItem> shoppingCartItems = new
				// HashSet<ShoppingCartItem>();
				for (ShoppingCartItem item : items) {
					LOGGER.debug("Populate item " + item.getId());
					getPopulatedItem(item, store);
					LOGGER.debug("Obsolete item ? " + item.isObsolete());
					if (item.isObsolete()) {
						cartIsObsolete = true;
					}
				}

				Set<ShoppingCartItem> refreshedItems = new HashSet<>(items);

				shoppingCart.setLineItems(refreshedItems);
				update(shoppingCart);

				if (cartIsObsolete) {
					shoppingCart.setObsolete(true);
				}
				return shoppingCart;
			}

		} catch (Exception e) {
			LOGGER.error(e.getMessage());
			throw new ServiceException(e);
		}

		return shoppingCart;

	}

	@Override
	public ShoppingCartItem populateShoppingCartItem(Product product, MerchantStore store) throws ServiceException {
		Validate.notNull(product, "Product should not be null");
		Validate.notNull(product.getMerchantStore(), "Product.merchantStore should not be null");
		Validate.notNull(store, "MerchantStore should not be null");

		ShoppingCartItem item = new ShoppingCartItem(product);
		item.setSku(product.getSku());//already in the constructor

		// set item price
		FinalPrice price = pricingService.calculateProductPrice(product);
		item.setItemPrice(price.getFinalPrice());
		return item;

	}

	@Transactional
	private void getPopulatedItem(final ShoppingCartItem item, MerchantStore store) throws Exception {

		Product product = productService.getBySku(item.getSku(), store, store.getDefaultLanguage());

		if (product == null) {
			item.setObsolete(true);
			return;
		}

		item.setProduct(product);
		item.setSku(product.getSku());

		if (product.isProductVirtual()) {
			item.setProductVirtual(true);
		}

		Set<ShoppingCartAttributeItem> cartAttributes = item.getAttributes();
		Set<ProductAttribute> productAttributes = product.getAttributes();
		List<ProductAttribute> attributesList = new ArrayList<ProductAttribute>();// attributes maintained
		List<ShoppingCartAttributeItem> removeAttributesList = new ArrayList<ShoppingCartAttributeItem>();// attributes
																											// to remove
		// DELETE ORPHEANS MANUALLY
		if ((productAttributes != null && productAttributes.size() > 0)
				|| (cartAttributes != null && cartAttributes.size() > 0)) {
			if (cartAttributes != null) {
				for (ShoppingCartAttributeItem attribute : cartAttributes) {
					long attributeId = attribute.getProductAttributeId();
					boolean existingAttribute = false;
					for (ProductAttribute productAttribute : productAttributes) {

						if (productAttribute.getId().equals(attributeId)) {
							attribute.setProductAttribute(productAttribute);
							attributesList.add(productAttribute);
							existingAttribute = true;
							break;
						}
					}

					if (!existingAttribute) {
						removeAttributesList.add(attribute);
					}

				}
			}
		}

		// cleanup orphean item
		if (CollectionUtils.isNotEmpty(removeAttributesList)) {
			for (ShoppingCartAttributeItem attr : removeAttributesList) {
				shoppingCartAttributeItemRepository.delete(attr);
			}
		}

		// cleanup detached attributes
		if (CollectionUtils.isEmpty(attributesList)) {
			item.setAttributes(null);
		}

		// set item price
		FinalPrice price = pricingService.calculateProductPrice(product, attributesList);
		item.setItemPrice(price.getFinalPrice());
		item.setFinalPrice(price);

		BigDecimal subTotal = item.getItemPrice().multiply(new BigDecimal(item.getQuantity()));
		item.setSubTotal(subTotal);

	}

	@Override
	public List<ShippingProduct> createShippingProduct(final ShoppingCart cart) throws ServiceException {
		/**
		 * Determines if products are virtual
		 */
		Set<ShoppingCartItem> items = cart.getLineItems();
		List<ShippingProduct> shippingProducts = null;
		for (ShoppingCartItem item : items) {
			Product product = item.getProduct();
			if (!product.isProductVirtual() && product.isProductShipeable()) {
				if (shippingProducts == null) {
					shippingProducts = new ArrayList<ShippingProduct>();
				}
				ShippingProduct shippingProduct = new ShippingProduct(product);
				shippingProduct.setQuantity(item.getQuantity());
				shippingProduct.setFinalPrice(item.getFinalPrice());
				shippingProducts.add(shippingProduct);
			}
		}

		return shippingProducts;

	}

	@Override
	public void removeShoppingCart(final ShoppingCart cart) throws ServiceException {
		shoppingCartRepository.delete(cart);
	}

	@Override
	public ShoppingCart mergeShoppingCarts(final ShoppingCart userShoppingModel, final ShoppingCart sessionCart,
			final MerchantStore store) throws Exception {
		if (sessionCart.getCustomerId() != null
				&& sessionCart.getCustomerId().equals(userShoppingModel.getCustomerId())) {
			LOGGER.info("Session Shopping cart belongs to same logged in user");
			if (CollectionUtils.isNotEmpty(userShoppingModel.getLineItems())
					&& CollectionUtils.isNotEmpty(sessionCart.getLineItems())) {
				return userShoppingModel;
			}
		}

		LOGGER.info("Starting merging shopping carts");
		if (CollectionUtils.isNotEmpty(sessionCart.getLineItems())) {
			Set<ShoppingCartItem> shoppingCartItemsSet = getShoppingCartItems(sessionCart, store, userShoppingModel);
			boolean duplicateFound = false;
			if (CollectionUtils.isNotEmpty(shoppingCartItemsSet)) {
				for (ShoppingCartItem sessionShoppingCartItem : shoppingCartItemsSet) {
					if (CollectionUtils.isNotEmpty(userShoppingModel.getLineItems())) {
						for (ShoppingCartItem cartItem : userShoppingModel.getLineItems()) {
							if (cartItem.getProduct().getId().longValue() == sessionShoppingCartItem.getProduct()
									.getId().longValue()) {
								if (CollectionUtils.isNotEmpty(cartItem.getAttributes())) {
									if (!duplicateFound) {
										LOGGER.info("Dupliate item found..updating exisitng product quantity");
										cartItem.setQuantity(
												cartItem.getQuantity() + sessionShoppingCartItem.getQuantity());
										duplicateFound = true;
										break;
									}
								}
							}
						}
					}
					if (!duplicateFound) {
						LOGGER.info("New item found..adding item to Shopping cart");
						userShoppingModel.getLineItems().add(sessionShoppingCartItem);
					}
				}

			}

		}
		LOGGER.info("Shopping Cart merged successfully.....");
		saveOrUpdate(userShoppingModel);
		removeShoppingCart(sessionCart);

		return userShoppingModel;
	}

	private Set<ShoppingCartItem> getShoppingCartItems(final ShoppingCart sessionCart, final MerchantStore store,
			final ShoppingCart cartModel) throws Exception {

		Set<ShoppingCartItem> shoppingCartItemsSet = null;
		if (CollectionUtils.isNotEmpty(sessionCart.getLineItems())) {
			shoppingCartItemsSet = new HashSet<ShoppingCartItem>();
			for (ShoppingCartItem shoppingCartItem : sessionCart.getLineItems()) {
				Product product = productService.getBySku(shoppingCartItem.getSku(), store, store.getDefaultLanguage());
						//.getById(shoppingCartItem.getProductId());
				if (product == null) {
					throw new Exception("Item with sku " + shoppingCartItem.getSku() + " does not exist");
				}

				if (product.getMerchantStore().getId().intValue() != store.getId().intValue()) {
					throw new Exception("Item with sku " + shoppingCartItem.getSku()
							+ " does not belong to merchant " + store.getId());
				}

				ShoppingCartItem item = populateShoppingCartItem(product, store);
				item.setQuantity(shoppingCartItem.getQuantity());
				item.setShoppingCart(cartModel);

				List<ShoppingCartAttributeItem> cartAttributes = new ArrayList<ShoppingCartAttributeItem>();
				if (shoppingCartItem != null && !CollectionUtils.isEmpty(shoppingCartItem.getAttributes())) {
					cartAttributes.addAll(shoppingCartItem.getAttributes());
					if (CollectionUtils.isNotEmpty(cartAttributes)) {
						for (ShoppingCartAttributeItem shoppingCartAttributeItem : cartAttributes) {
							ProductAttribute productAttribute = productAttributeService
									.getById(shoppingCartAttributeItem.getId());
							if (productAttribute != null && productAttribute.getProduct().getId().longValue() == product
									.getId().longValue()) {

								ShoppingCartAttributeItem attributeItem = new ShoppingCartAttributeItem(item,
										productAttribute);
								if (shoppingCartAttributeItem.getId() > 0) {
									attributeItem.setId(shoppingCartAttributeItem.getId());
								}
								item.addAttributes(attributeItem);

							}
						}
					}
				}

				shoppingCartItemsSet.add(item);
			}

		}
		return shoppingCartItemsSet;
	}

	@Override
	@Transactional
	public void deleteShoppingCartItem(Long id) {

		ShoppingCartItem item = shoppingCartItemRepository.findOne(id);
		if (item != null) {

			if (item.getAttributes() != null) {
				item.getAttributes().forEach(a -> shoppingCartAttributeItemRepository.deleteById(a.getId()));
				item.getAttributes().clear();
			}

			// refresh
			item = shoppingCartItemRepository.findOne(id);

			// delete
			shoppingCartItemRepository.deleteById(id);

		}

	}

}



```
