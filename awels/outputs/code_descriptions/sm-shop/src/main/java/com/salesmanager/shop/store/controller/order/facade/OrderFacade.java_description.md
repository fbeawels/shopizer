# OrderFacade.java

## Review

## 1. Summary  

**Purpose**  
`OrderFacade` is the central service contract for all order‑related business logic in the shop module.  
It encapsulates the orchestration of order creation, validation, pricing, shipping, payment, status handling, and reporting.  

**Key components**  

| Responsibility | Interface method(s) | Notes |
|----------------|---------------------|-------|
| Order initialization | `initializeOrder`, `refreshOrder` | Build a new `ShopOrder` from a cart and customer data. |
| Pricing & totals | `calculateOrderTotal` (two overloads) | Returns an `OrderTotalSummary` for web or API callers. |
| Order processing | `processOrder` (three overloads) | Persists the order, applies transaction data and creates the final `Order`. |
| Shipping | `getShippingQuote` (three overloads), `getShippingSummary` | Computes quotes and summary objects. |
| Validation | `validateOrder` | Uses Spring’s `BindingResult` and a messages map. |
| Order retrieval | `getReadableOrder`, `getReadableOrderList` (four overloads), `getReadableOrderHistory` | Returns DTOs for UI or API. |
| Payment capture | `captureOrder`, `nextTransaction`, `listTransactions` | Supports pre‑authorization workflows. |
| Order status | `createOrderStatus`, `updateOrderStatus`, `updateOrderCustomre` | Tracks status changes and history. |
| Utility | `initEmptyCustomer`, `getShipToCountry` | Helper methods for anonymous users and shipping country lists. |

**Design patterns / frameworks**  
* **Facade Pattern** – hides complex order workflows behind a simple API.  
* **Service Layer** – business logic is exposed through an interface to be implemented by a Spring component.  
* **DTO / VO** – uses `ShopOrder`, `PersistableOrder`, `ReadableOrder`, etc. to separate persistence from presentation.  
* **Spring Validation** – `BindingResult` integration.  
* **Exception handling** – throws `Exception` or `ServiceException` for domain‑specific errors.

---

## 2. Detailed Description  

### Core Flow

1. **Order Creation**  
   * `initializeOrder` builds a new `ShopOrder` from a `ShoppingCart`, `Customer` and `Language`.  
   * `refreshOrder` allows a previously created order to be updated with new cart or customer data.  

2. **Pricing**  
   * `calculateOrderTotal` computes taxes, shipping, discounts and returns an `OrderTotalSummary`.  
   * Overloads exist for the web UI (`ShopOrder`) and the REST API (`PersistableOrder`).  

3. **Validation**  
   * `validateOrder` populates a `BindingResult` with business rules and returns localized messages in a map.  

4. **Processing**  
   * `processOrder` (three overloads) persists the `Order` entity.  
   * The API version can accept a `Locale` for i18n of order comments or other fields.  

5. **Shipping**  
   * `getShippingQuote` returns a `ShippingQuote` based on cart contents, customer and store configuration.  
   * `getShippingSummary` transforms a `ShippingQuote` into a `ShippingSummary` that can be merged into the total.  

6. **Payment Capture**  
   * `getCapturableOrderList` returns orders ready for capture.  
   * `captureOrder` performs the capture and returns a `ReadableTransaction`.  

7. **Status Management**  
   * `createOrderStatus` and `updateOrderStatus` manage the `order_status_history`.  
   * `nextTransaction` hints at the next expected payment step (e.g., authorize → capture).  

8. **Reporting / Retrieval**  
   * `getReadableOrder`, `getReadableOrderList`, and `getReadableOrderHistory` provide DTOs for display or API consumption.  

### Assumptions & Constraints  

* **Thread‑safety** – Implementations must be stateless or properly synchronized because the interface may be used concurrently.  
* **Locale handling** – Some methods accept a `Locale` (API overload), but others rely on `Language`. Implementers must keep these in sync.  
* **Exception strategy** – Most methods declare `throws Exception`; this is too generic. The `ServiceException` should be the standard domain exception.  
* **Customer anonymity** – `initEmptyCustomer` returns a customer stub for guest checkout.  
* **Country list** – `getShipToCountry` is tied to the store’s configuration and should return an immutable list.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects / Notes |
|--------|---------|--------|---------|---------------------|
| `initializeOrder` | Build a new order from cart/customer data. | `MerchantStore`, `Customer`, `ShoppingCart`, `Language` | `ShopOrder` | Throws if data invalid. |
| `refreshOrder` | Re‑calculate totals & line items on an existing order. | `ShopOrder`, `MerchantStore`, `Customer`, `ShoppingCart`, `Language` | void | Persists changes internally. |
| `calculateOrderTotal` (web) | Compute total price for `ShopOrder`. | `MerchantStore`, `ShopOrder`, `Language` | `OrderTotalSummary` | Handles taxes, discounts. |
| `calculateOrderTotal` (API) | Same but for `PersistableOrder`. | `MerchantStore`, `PersistableOrder`, `Language` | `OrderTotalSummary` | Allows API callers to preview totals. |
| `processOrder` (web) | Persist a validated order. | `ShopOrder`, `Customer`, `MerchantStore`, `Language` | `Order` | Creates Order entity and related history. |
| `processOrder` (with transaction) | Persist an order and link to an existing transaction. | `ShopOrder`, `Customer`, `Transaction`, `MerchantStore`, `Language` | `Order` | Used for pre‑authorized orders. |
| `processOrder` (API v1) | Persist an order from API v1 model. | `PersistableOrder`, `Customer`, `MerchantStore`, `Language`, `Locale` | `Order` | Supports localized fields. |
| `initEmptyCustomer` | Returns a blank customer for guest checkout. | `MerchantStore` | `Customer` | Fields may be null. |
| `getShipToCountry` | List all countries a store can ship to. | `MerchantStore`, `Language` | `List<Country>` | Uses store config. |
| `getShippingQuote` (3 overloads) | Compute shipping quote for given cart/order. | See overload signatures | `ShippingQuote` | Handles customer address, store rules. |
| `getShippingSummary` | Convert a quote into a summary for total calculation. | `ShippingQuote`, `MerchantStore`, `Language` | `ShippingSummary` | Combines quote details. |
| `validateOrder` | Apply business validation rules. | `ShopOrder`, `BindingResult`, `Map<String,String>`, `MerchantStore`, `Locale` | void | Fills binding errors and messages map. |
| `getReadableOrder` | Retrieve an order DTO for a given ID. | `Long`, `MerchantStore`, `Language` | `ReadableOrder` | Throws if not found. |
| `getReadableOrderHistory` | List status history for an order. | `Long`, `MerchantStore`, `Language` | `List<ReadableOrderStatusHistory>` | |
| `getReadableOrderList` (4 overloads) | Paginated or criteria‑based listing of orders. | See overload signatures | `ReadableOrderList` | Returns DTOs. |
| `getCapturableOrderList` | List orders pending capture. | `MerchantStore`, `Date`, `Date`, `Language` | `ReadableOrderList` | |
| `captureOrder` | Perform capture on a pre‑authorized transaction. | `MerchantStore`, `Order`, `Customer`, `Language` | `ReadableTransaction` | |
| `nextTransaction` | Return the next expected transaction type for an order. | `Long`, `MerchantStore` | `TransactionType` | |
| `createOrderStatus` | Add a new status record to history. | `PersistableOrderStatusHistory`, `Long`, `MerchantStore` | void | |
| `updateOrderCustomre` | Update order’s customer fields (typo in name). | `Long`, `PersistableCustomer`, `MerchantStore` | void | |
| `listTransactions` | Return all transactions for an order. | `Long`, `MerchantStore` | `List<ReadableTransaction>` | |
| `updateOrderStatus` | Change status and create history record. | `Order`, `OrderStatus`, `MerchantStore` | void | |

---

## 4. Dependencies  

| Category | Dependency | Notes |
|----------|------------|-------|
| **Core domain** | `com.salesmanager.core.model.*` | All business entities (Order, Customer, ShippingQuote, etc.). |
| **DTOs** | `com.salesmanager.shop.model.*` | Presentation objects. |
| **Spring Validation** | `org.springframework.validation.BindingResult` | For binding validation errors. |
| **Exception handling** | `com.salesmanager.core.business.exception.ServiceException` | Domain‑specific service error. |
| **Date/Locale** | `java.util.Date`, `java.util.Locale` | Standard Java. |
| **Collections** | `java.util.List`, `java.util.Map` | Standard Java. |
| **No explicit Spring annotations** – as an interface, no component scanning here, but the implementation will typically be a Spring `@Service`. |

All dependencies are third‑party domain objects; none are platform‑specific beyond Java and Spring.

---

## 5. Additional Notes & Recommendations  

### 1. Naming & Typos  
* `updateOrderCustomre` – typo; should be `updateOrderCustomer`.  
* Consistency between `PersistableOrderStatusHistory` and `ReadableOrderStatusHistory` is good; however, ensure that the names clearly differentiate between DTOs used for input vs output.

### 2. Exception Strategy  
* The interface mixes `throws Exception` and `throws ServiceException`. Prefer a single unchecked or checked domain exception (`ServiceException`) for all failures to simplify error handling for callers.

### 3. Overloading Clarity  
* Several overloaded `calculateOrderTotal` and `getShippingQuote` methods could be confusing. Consider using descriptive parameter names or separate service interfaces for web vs API flows.

### 4. Locale & Language Handling  
* The API methods accept both `Locale` and `Language`. Document the conversion strategy and ensure that localization of messages, taxes, and shipping rules remains consistent.

### 5. Thread Safety & Statelessness  
* The facade is expected to be stateless. Explicitly document this expectation and provide guidelines for thread‑safe caching if needed.

### 6. DTO Design  
* `ReadableOrderList` appears to be a paginated wrapper; ensure it includes pagination metadata (`total`, `page`, `size`) to aid API consumers.

### 7. Validation Flow  
* `validateOrder` populates both a `BindingResult` and a `messagesResult` map. The dual mechanism can be unified: use a single source of validation errors (e.g., a `ValidationResult` object).

### 8. Future Enhancements  
* **Pagination & Sorting** – Add optional `Sort` parameters to listing methods.  
* **Event Publication** – Emit domain events (e.g., `OrderPlaced`, `OrderShipped`) for asynchronous integrations.  
* **Audit Logging** – Provide hooks to log critical order changes.  
* **Transactional Boundaries** – Clarify which methods should run within a transaction (likely all `processOrder` and status update methods).  

### 9. Edge Cases  
* **Pre‑authorization Expiry** – `captureOrder` should handle cases where the auth window has expired.  
* **Duplicate Order Prevention** – Ensure `processOrder` checks for duplicate order IDs when called from external APIs.  
* **Shipping Availability** – `getShippingQuote` may return `null`; callers should handle this gracefully.  

---

### Verdict  

`OrderFacade` provides a comprehensive contract for order lifecycle management, neatly separating concerns and exposing a clean API to higher layers (web controllers, REST endpoints, background jobs). The interface is well‑documented, but could benefit from a stricter exception model, typo corrections, and clearer method signatures. With these refinements, the facade will serve as a robust backbone for the shop’s order processing subsystem.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.order.facade;

import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import com.salesmanager.core.model.order.orderstatus.OrderStatus;
import org.springframework.validation.BindingResult;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.order.OrderCriteria;
import com.salesmanager.core.model.order.OrderTotalSummary;
import com.salesmanager.core.model.payments.Transaction;
import com.salesmanager.core.model.payments.TransactionType;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.shipping.ShippingSummary;
import com.salesmanager.core.model.shoppingcart.ShoppingCart;
import com.salesmanager.shop.model.customer.PersistableCustomer;
import com.salesmanager.shop.model.order.ShopOrder;
import com.salesmanager.shop.model.order.history.PersistableOrderStatusHistory;
import com.salesmanager.shop.model.order.history.ReadableOrderStatusHistory;
import com.salesmanager.shop.model.order.transaction.ReadableTransaction;


public interface OrderFacade {

	ShopOrder initializeOrder(MerchantStore store, Customer customer, ShoppingCart shoppingCart, Language language) throws Exception;
	void refreshOrder(ShopOrder order, MerchantStore store, Customer customer, ShoppingCart shoppingCart, Language language) throws Exception;
	/** used in website **/
	OrderTotalSummary calculateOrderTotal(MerchantStore store, ShopOrder order, Language language) throws Exception;
	/** used in the API **/
	OrderTotalSummary calculateOrderTotal(MerchantStore store, com.salesmanager.shop.model.order.v0.PersistableOrder order, Language language) throws Exception;

	/** process a valid order **/
	Order processOrder(ShopOrder order, Customer customer, MerchantStore store, Language language) throws ServiceException;
	/** process a valid order against an initial transaction **/
	Order processOrder(ShopOrder order, Customer customer, Transaction transaction, MerchantStore store, Language language) throws ServiceException;
	/** process a valid order submitted from the API **/
	Order processOrder(com.salesmanager.shop.model.order.v1.PersistableOrder order, Customer customer, MerchantStore store, Language language, Locale locale) throws ServiceException;



	/** creates a working copy of customer when the user is anonymous **/
	Customer initEmptyCustomer(MerchantStore store);
	List<Country> getShipToCountry(MerchantStore store, Language language)
			throws Exception;

	/**
	 * Get a ShippingQuote based on merchant configuration and items to be shipped
	 * @param cart
	 * @param order
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ShippingQuote getShippingQuote(PersistableCustomer customer, ShoppingCart cart, ShopOrder order,
			MerchantStore store, Language language) throws Exception;

	ShippingQuote getShippingQuote(Customer customer, ShoppingCart cart, com.salesmanager.shop.model.order.v0.PersistableOrder order,
			MerchantStore store, Language language) throws Exception;

	ShippingQuote getShippingQuote(Customer customer, ShoppingCart cart,
			MerchantStore store, Language language) throws Exception;

	/**
	 * Creates a ShippingSummary object for OrderTotal calculation based on a ShippingQuote
	 * @param quote
	 * @param store
	 * @param language
	 * @return
	 */
	ShippingSummary getShippingSummary(ShippingQuote quote, MerchantStore store, Language language);

	/**
	 * Validates an order submitted from the web application
	 * @param order
	 * @param bindingResult
	 * @param messagesResult
	 * @param store
	 * @param locale
	 * @throws ServiceException
	 */
	void validateOrder(ShopOrder order, BindingResult bindingResult,
			Map<String, String> messagesResult, MerchantStore store,
			Locale locale) throws ServiceException;

	/**
	 * Creates a ReadableOrder object from an orderId
	 * @param orderId
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	com.salesmanager.shop.model.order.v0.ReadableOrder getReadableOrder(Long orderId, MerchantStore store, Language language);

	/**
	 * List of orderstatus history
	 * @param orderId
	 * @param store
	 * @param language
	 * @return
	 */
	List<ReadableOrderStatusHistory> getReadableOrderHistory(Long orderId, MerchantStore store, Language language);


	/**
     * <p>Method used to fetch all orders associated with customer customer.
     * It will used current customer ID to fetch all orders which has been
     * placed by customer for current store.</p>
     *
     * @param customer currently logged in customer
     * @param store store associated with current customer
     * @return ReadableOrderList
     * @throws Exception
     */

	com.salesmanager.shop.model.order.v0.ReadableOrderList getReadableOrderList(MerchantStore store, Customer customer, int start,
			int maxCount, Language language) throws Exception;


	/**
	 * <p>Method used to fetch all orders associated with customer customer.
	 * It will used current customer ID to fetch all orders which has been
	 * placed by customer for current store.</p>
	 *
	 * @return ReadableOrderList
	 * @throws Exception
	 */

	com.salesmanager.shop.model.order.v0.ReadableOrderList getReadableOrderList(OrderCriteria criteria, MerchantStore store);


	/**
	 * Get a list of Order on which payment capture must be done
	 * @param store
	 * @param startDate
	 * @param endDate
	 * @param language
	 * @return
	 * @throws Exception
	 */
	com.salesmanager.shop.model.order.v0.ReadableOrderList getCapturableOrderList(MerchantStore store, Date startDate, Date endDate,
			Language language) throws Exception;

	/**
	 * Capture a pre-authorized transaction. Candidate order ids can be obtained from
	 * getCapturableOrderList
	 * @param store
	 * @param order
	 * @param customer
	 * @return
	 * @throws Exception
	 */
	ReadableTransaction captureOrder(MerchantStore store, Order order, Customer customer, Language language) throws Exception;

	/**
	 * Returns next TransactionType expected if any.
	 */
	TransactionType nextTransaction(Long orderId, MerchantStore store);

	/**
	 * Get orders for a given store
	 * @param store
	 * @param start
	 * @param maxCount
	 * @param language
	 * @return
	 * @throws Exception
	 */
	com.salesmanager.shop.model.order.v0.ReadableOrderList getReadableOrderList(MerchantStore store, int start,
			int maxCount, Language language) throws Exception;

	/**
	 * Adds a status to an order status history
	 * @param status
	 * @param id
	 * @param store
	 */
	void createOrderStatus(PersistableOrderStatusHistory status, Long id, MerchantStore store);

	/**
	 * Updates order customer
	 * Only updates customer information from the order
	 * It won't update customer object from Customer entity
	 * @param orderId
	 * @param customer
	 * @param store
	 */
	void updateOrderCustomre(Long orderId, PersistableCustomer customer, MerchantStore store);

	List<ReadableTransaction> listTransactions (Long orderId, MerchantStore store);

	/**
	 * Update Order status and create order_status_history record
	 */
	void updateOrderStatus(Order order, OrderStatus newStatus, MerchantStore store);
}



```
