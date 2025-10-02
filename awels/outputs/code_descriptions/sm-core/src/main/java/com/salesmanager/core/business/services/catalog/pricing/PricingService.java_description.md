# PricingService.java

## Review

## 1. Summary  

**Purpose**  
`PricingService` is a pure‑Java interface that declares the contract for all pricing‑related operations within the SalesManager core catalog. It abstracts the logic needed to:

- Compute the final price of a product, variant, or specific inventory item.
- Apply customer‑specific discounts, rebate logic, and attribute‑based price adjustments.
- Format monetary values for presentation (including locale, currency, and store‑specific formatting).
- Convert user‑supplied string amounts into validated `BigDecimal` values.

**Key Components**

| Component | Role |
|-----------|------|
| `calculateProductPrice` (several overloads) | Central price calculation entry points for different contexts (product, variant, availability, attributes, customer). |
| `getDisplayAmount` | Produces a user‑friendly string that includes the currency symbol, proper decimal formatting, and store‑specific conventions. |
| `getAmount` | Parses and validates a string representation of money into a `BigDecimal`. |
| `getStringAmount` | Returns a plain number string (no currency symbol) for use in API responses or logs. |
| `calculatePriceQuantity` | Simple helper to multiply a unit price by a quantity, used when computing order totals. |

**Design Patterns / Libraries**

- **Interface Segregation**: The interface defines only pricing operations, allowing different implementations (e.g., in‑memory, database‑backed, or external pricing services) to be swapped in via dependency injection.
- **Strategy Pattern**: Clients can choose among the overloaded `calculateProductPrice` methods to invoke the appropriate pricing strategy (product‑level, variant‑level, attribute‑level, customer‑specific).
- **Java Standard Library**: Uses `BigDecimal` for monetary precision, `Locale` for i18n formatting, and the custom domain classes (`Product`, `ProductVariant`, etc.) from SalesManager.

---

## 2. Detailed Description  

### Core Interaction Flow  

1. **Initialization**  
   - In production, a concrete implementation (e.g., `PricingServiceImpl`) is injected into services/controllers via a DI framework (Spring, CDI, etc.).  
   - The implementation typically loads static price data from the database (e.g., `ProductPrice`, `VariantPrice`, `CustomerRebate`) and caches it for performance.

2. **Runtime Behavior**  
   - When a product or variant is requested (e.g., from a product catalog or shopping cart), the client calls one of the `calculateProductPrice` overloads.  
   - The implementation:
     - Retrieves base price(s) from the product entity or its variants/availability records.  
     - Applies any customer‑specific discounts or loyalty rebates.  
     - Adjusts price for selected attributes (e.g., size, color) that have differential pricing.  
     - Returns a `FinalPrice` object that contains the computed amount, currency, tax, and any applied discounts.

3. **Formatting**  
   - The client may ask for a display string via `getDisplayAmount` or `getStringAmount`.  
   - The service uses the provided `MerchantStore` and `Locale` to format the amount correctly (currency symbol, grouping separators, decimal places).

4. **Cleanup**  
   - No explicit cleanup is required; the interface does not hold state.  
   - If an implementation caches data, it may provide a method for cache eviction (not part of this interface).

### Assumptions & Constraints  

- **Immutability of Domain Objects**: It is assumed that `Product`, `ProductVariant`, `ProductAvailability`, and `Customer` are immutable or are only read during calculation.  
- **Non‑negative Quantities**: `calculatePriceQuantity` does not guard against negative or zero values; callers must ensure validity.  
- **Currency Consistency**: All monetary operations presume that the currency of the product and the store match; mismatches are not handled here.  
- **Exception Semantics**: Every method throws `ServiceException`, implying that any domain or data‑access errors should be wrapped into this unchecked exception.

### Architectural Choices  

- **Method Overloading vs. Strategy Objects**: The interface opts for overloaded methods rather than a single strategy interface. While this keeps the API simple for clients, it can lead to ambiguity when the same argument types are used (e.g., `calculateProductPrice(Product, Customer)` vs. `calculateProductPrice(Product, List<ProductAttribute>)`).  
- **Granular Pricing Methods**: Separating price calculation by entity type (product, variant, availability) clarifies intent but increases API surface area.  
- **Utility Methods**: `calculatePriceQuantity`, `getDisplayAmount`, and `getStringAmount` are convenience methods that reduce boilerplate in client code.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return Type | Exceptions |
|--------|---------|------------|-------------|------------|
| `calculateProductPrice(Product)` | Compute final price of a product using its default prices. | `Product product` | `FinalPrice` | `ServiceException` |
| `calculateProductPrice(ProductVariant)` | Compute price for a variant (may use variant‑specific inventory). | `ProductVariant variant` | `FinalPrice` | `ServiceException` |
| `calculateProductPrice(ProductAvailability)` | Compute price for a specific inventory record. | `ProductAvailability product` | `FinalPrice` | `ServiceException` |
| `calculateProductPrice(Product, Customer)` | Compute price with customer‑specific discounts. | `Product product`, `Customer customer` | `FinalPrice` | `ServiceException` |
| `calculateProductPrice(Product, List<ProductAttribute>)` | Compute price with attribute‑based adjustments. | `Product product`, `List<ProductAttribute> attributes` | `FinalPrice` | `ServiceException` |
| `calculateProductPrice(Product, List<ProductAttribute>, Customer)` | Combine attribute adjustments and customer discounts. | `Product product`, `List<ProductAttribute> attributes`, `Customer customer` | `FinalPrice` | `ServiceException` |
| `getDisplayAmount(BigDecimal, MerchantStore)` | Format amount with store’s currency symbol and locale. | `BigDecimal amount`, `MerchantStore store` | `String` | `ServiceException` |
| `getDisplayAmount(BigDecimal, Locale, Currency, MerchantStore)` | More fine‑grained formatting: locale, currency, store. | `BigDecimal amount`, `Locale locale`, `Currency currency`, `MerchantStore store` | `String` | `ServiceException` |
| `getAmount(String)` | Parse a string into a validated `BigDecimal`. | `String amount` | `BigDecimal` | `ServiceException` |
| `getStringAmount(BigDecimal, MerchantStore)` | Return raw numeric string without currency symbol. | `BigDecimal amount`, `MerchantStore store` | `String` | `ServiceException` |
| `calculatePriceQuantity(BigDecimal, int)` | Multiply unit price by quantity. | `BigDecimal price`, `int quantity` | `BigDecimal` | – |

**Reusable / Utility Methods**  
- `calculatePriceQuantity` is a pure arithmetic helper that can be reused anywhere a line‑item total is needed.  
- The two `getDisplayAmount` overloads centralize locale‑aware formatting logic, ensuring consistency across the UI.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | Standard JDK | Used for monetary values. |
| `java.util.Locale` | Standard JDK | For i18n formatting. |
| `java.util.List` | Standard JDK | For collections of attributes. |
| `com.salesmanager.core.business.exception.ServiceException` | Custom | Wraps all runtime exceptions in the service layer. |
| Domain classes (`Product`, `ProductVariant`, `ProductAvailability`, `ProductAttribute`, `Customer`, `MerchantStore`, `Currency`, `FinalPrice`) | Custom | Represent business entities. No external libraries are referenced. |
| **No external frameworks** (e.g., Spring, Hibernate) are declared here; they would be added by concrete implementations. |

---

## 5. Additional Notes  

### Edge Cases & Potential Improvements  

1. **Null Handling**  
   - The interface documentation does not specify how `null` arguments are treated. Implementations should either throw `IllegalArgumentException` or document expected behavior.  
   - In particular, `calculatePriceQuantity` should guard against negative `quantity` values.

2. **Currency Mismatch**  
   - Methods like `calculateProductPrice(Product)` implicitly assume that the product’s currency matches the store’s currency. If prices are stored in multiple currencies, a conversion step or explicit currency parameter would be needed.

3. **Method Overload Ambiguity**  
   - Overloads that differ only by parameter count (e.g., `calculateProductPrice(Product, Customer)` vs. `calculateProductPrice(Product, List<ProductAttribute>)`) can be confusing. Consider using a builder pattern or a dedicated request object (`PricingRequest`) to encapsulate all optional parameters.

4. **Performance & Caching**  
   - Frequent calls to `calculateProductPrice` could lead to repeated DB lookups. A caching strategy (e.g., Guava Cache, Spring Cache) should be employed in the implementation.

5. **Internationalization**  
   - `getDisplayAmount(BigDecimal, MerchantStore)` relies on the store’s locale. If the user’s locale differs from the store locale, a mismatch may occur. It may be preferable to expose a `getDisplayAmount(BigDecimal, Locale, Currency, MerchantStore)` overload to the caller.

6. **Testing**  
   - Unit tests should mock the domain objects and verify that each overload behaves correctly. Edge cases such as zero quantity, null attributes, and large monetary values should be covered.

7. **Future Enhancements**  
   - **Dynamic Pricing**: Integrate time‑based promotions or flash sales.  
   - **Tax Calculation**: Add a method that returns tax‑inclusive and tax‑exclusive prices.  
   - **Bulk Discount Engine**: Support tiered pricing based on quantity ranges.  
   - **Event‑Driven Updates**: Publish events when pricing changes so that dependent caches can refresh.

### Overall Impression  

The `PricingService` interface is well‑structured for a typical e‑commerce platform. It cleanly separates concerns: calculation logic, formatting, and parsing. The use of an interface allows multiple, testable implementations. With the small refinements above—especially around overload clarity, null safety, and currency handling—the contract becomes even more robust and developer‑friendly.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.pricing;

import java.math.BigDecimal;
import java.util.List;
import java.util.Locale;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.currency.Currency;


/**
 * Services for Product item price calculation. 
 * @author Carl Samson
 *
 */
public interface PricingService {

	/**
	 * Calculates the FinalPrice of a Product taking into account
	 * all defined prices and possible rebates
	 * @param product
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(Product product) throws ServiceException;
	
	/**
	 * Calculates variant price specific to variant inventory or parent inventory
	 * @param variant
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(ProductVariant variant) throws ServiceException;
	
	/**
	 * Calculates the price on a specific inventory
	 * @param product
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(ProductAvailability product) throws ServiceException;

	/**
	 * Calculates the FinalPrice of a Product taking into account
	 * all defined prices and possible rebates. It also applies other calculation
	 * based on the customer
	 * @param product
	 * @param customer
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(Product product, Customer customer)
			throws ServiceException;

	/**
	 * Calculates the FinalPrice of a Product taking into account
	 * all defined prices and possible rebates. This method should be used to calculate
	 * any additional prices based on the default attributes or based on the user selected attributes.
	 * @param product
	 * @param attributes
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(Product product,
			List<ProductAttribute> attributes) throws ServiceException;

	/**
	 * Calculates the FinalPrice of a Product taking into account
	 * all defined prices and possible rebates. This method should be used to calculate
	 * any additional prices based on the default attributes or based on the user selected attributes.
	 * It also applies other calculation based on the customer
	 * @param product
	 * @param attributes
	 * @param customer
	 * @return
	 * @throws ServiceException
	 */
	FinalPrice calculateProductPrice(Product product,
			List<ProductAttribute> attributes, Customer customer)
			throws ServiceException;

	/**
	 * Method to be used to print a displayable formated amount to the end user
	 * @param amount
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	String getDisplayAmount(BigDecimal amount, MerchantStore store)
			throws ServiceException;
	
	/**
	 * Method to be used when building an amount formatted with the appropriate currency
	 * @param amount
	 * @param locale
	 * @param currency
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	String getDisplayAmount(BigDecimal amount, Locale locale, Currency currency, MerchantStore store)
			throws ServiceException;
	
	/**
	 * Converts a String amount to BigDecimal
	 * Takes care of String amount validation
	 * @param amount
	 * @return
	 * @throws ServiceException
	 */
	BigDecimal getAmount(String amount) throws ServiceException;
	
	/**
	 * String format of the money amount without currency symbol
	 * @param amount
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	String getStringAmount(BigDecimal amount, MerchantStore store)
			throws ServiceException;

	/**
	 * Method for calculating sub total
	 * @param price
	 * @param quantity
	 * @return
	 */
	BigDecimal calculatePriceQuantity(BigDecimal price, int quantity);
}



```
