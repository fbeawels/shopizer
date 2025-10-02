# PricingServiceImpl.java

## Review

## 1. Summary

The **`PricingServiceImpl`** class is a Spring‐managed service that centralises all price‑calculation logic for catalog products.  
It delegates the heavy lifting to a utility class (`ProductPriceUtils`) and simply provides a façade that:

1. Calculates the final price of a product (with optional customer or attribute context).  
2. Formats amounts for display to the store or admin.  
3. Handles quantity‑based price multiplication.

Key components:
- **`ProductPriceUtils`** – contains the core algorithm for computing final prices.  
- **`FinalPrice`** – DTO that carries calculated amounts, discounts, taxes, etc.  
- **`ServiceException`** – domain‑specific exception wrapped around low‑level failures.  

The class uses the Spring `@Service` stereotype, SLF4J for logging, and standard Java APIs for arithmetic and locale handling.

---

## 2. Detailed Description

### Core Flow

| Step | Action | Notes |
|------|--------|-------|
| **Initialization** | Spring injects a single instance of `ProductPriceUtils` into the service (`@Inject`). | No explicit constructor; relies on default constructor + field injection. |
| **Price Calculation** | Each `calculateProductPrice…` method forwards the request to `priceUtil`, optionally passing a customer or attribute list. | The `priceUtil` implementation decides how to combine product data, attributes, customer tier, taxes, etc. |
| **Quantity Handling** | `calculatePriceQuantity` multiplies a `BigDecimal` price by a `BigDecimal` constructed from `int quantity`. | No rounding or scale adjustments. |
| **Formatting** | `getDisplayAmount`, `getStringAmount`, `getAmount` call corresponding `priceUtil` methods, wrap any `Exception` in `ServiceException`. | Logging is performed before re‑throwing. |
| **Variant Handling** | `calculateProductPrice(ProductVariant variant)` is currently unimplemented, returning `null`. | This is a clear bug and a missing feature. |
| **Cleanup** | None – the service is stateless; no resources are released. |

### Assumptions & Constraints

- **Statelessness**: The service does not keep any state; thread‑safety is assumed via immutable utilities.  
- **Null Safety**: No null checks on parameters; passing `null` will propagate to `priceUtil` and likely cause `NullPointerException`.  
- **Locale/Currency**: Relies on the underlying `priceUtil` to handle locale/currency formatting; the service only forwards the arguments.  
- **Error Handling**: Generic `Exception` catching masks the actual failure cause.  
- **Business Rules**: Customer‑specific pricing rules are noted as TODOs – the service currently ignores them.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `calculateProductPrice(Product)` | `FinalPrice` | Basic price calculation for a product. | `Product` | `FinalPrice` | None |
| `calculateProductPrice(Product, Customer)` | `FinalPrice` | Price calculation incorporating a customer. | `Product`, `Customer` | `FinalPrice` | None (TODO) |
| `calculateProductPrice(Product, List<ProductAttribute>)` | `FinalPrice` | Price with attribute overrides. | `Product`, `List<ProductAttribute>` | `FinalPrice` | None |
| `calculateProductPrice(Product, List<ProductAttribute>, Customer)` | `FinalPrice` | Price with attributes & customer. | `Product`, `List<ProductAttribute>`, `Customer` | `FinalPrice` | None (TODO) |
| `calculatePriceQuantity(BigDecimal, int)` | `BigDecimal` | Multiplies price by quantity. | `price`, `quantity` | `price * quantity` | None |
| `getDisplayAmount(BigDecimal, MerchantStore)` | `String` | Store‑localised currency string. | `amount`, `store` | Formatted string | Logs errors |
| `getDisplayAmount(BigDecimal, Locale, Currency, MerchantStore)` | `String` | Locale‑specific formatted amount. | `amount`, `locale`, `currency`, `store` | Formatted string | Logs errors |
| `getStringAmount(BigDecimal, MerchantStore)` | `String` | Admin‑display string. | `amount`, `store` | Formatted string | Logs errors |
| `getAmount(String)` | `BigDecimal` | Parse string to numeric amount. | `amount` | `BigDecimal` | Logs errors |
| `calculateProductPrice(ProductAvailability)` | `FinalPrice` | Price based on availability entity. | `availability` | `FinalPrice` | None |
| `calculateProductPrice(ProductVariant)` | `FinalPrice` | Price for a variant. | `variant` | **`null`** (bug) | None |

**Reusable/Utility Methods**  
- `calculatePriceQuantity` is a simple arithmetic helper; could be extracted into a `MathUtils` class if used elsewhere.

---

## 4. Dependencies

| Library / Framework | Usage | Third‑Party / Standard |
|---------------------|-------|------------------------|
| Spring Framework (`org.springframework.stereotype.Service`, `@Inject`) | Dependency injection & component scanning | Third‑Party |
| SLF4J (`org.slf4j.Logger`, `LoggerFactory`) | Logging | Third‑Party |
| Java Standard Library | `BigDecimal`, `List`, `Locale`, `Currency`, exception handling | Standard |
| `ProductPriceUtils`, `ServiceException`, domain models (`Product`, `Customer`, etc.) | Domain logic & DTOs | Project‑specific |

There are **no platform‑specific** dependencies; the code is portable across any JVM environment that supports Java 8+ (the use of `new BigDecimal(quantity)` is Java‑standard).

---

## 5. Additional Notes

### Strengths
- **Clear Separation of Concerns**: Business logic lives in `ProductPriceUtils`; the service is a thin façade.  
- **Dependency Injection**: Easy to swap the utility with a mock for unit testing.  
- **Consistent Error Handling**: Wraps all failures in `ServiceException`.

### Weaknesses / Edge Cases
1. **Unimplemented Variant Pricing**  
   Returning `null` from `calculateProductPrice(ProductVariant)` will cause `NullPointerException`s downstream.  
2. **No Null Validation**  
   All public methods accept raw arguments without validation; a `null` product or attribute list will bubble up to `priceUtil` and likely crash.  
3. **Generic Exception Catching**  
   The `try/catch` blocks swallow all exceptions, log only a generic message, and re‑throw a generic `ServiceException`.  This obscures the root cause and hampers debugging.  
4. **Quantity Multiplication**  
   `new BigDecimal(quantity)` is unnecessarily expensive; `BigDecimal.valueOf(quantity)` would be more efficient and safer (handles large ints).  No scale/rounding is applied, which could be problematic for currency calculations.  
5. **Locale / Currency Formatting**  
   The service trusts `priceUtil` to format amounts correctly. If `priceUtil` uses `DecimalFormat`, locale‑specific grouping and decimal separators need careful handling.  
6. **Lack of Business Rules**  
   Customer‑specific or attribute‑specific pricing rules are marked as TODOs, indicating incomplete business logic.

### Suggested Enhancements
- **Implement Variant Pricing** or throw an explicit `UnsupportedOperationException` until fully implemented.  
- **Add Parameter Validation** (e.g., `Objects.requireNonNull`) to surface NPEs early.  
- **Refine Exception Handling**: Catch specific checked exceptions (`ParseException`, `NumberFormatException`) and wrap them; optionally expose the cause in the log.  
- **Use `BigDecimal.valueOf` for quantity**; consider rounding mode if necessary.  
- **Introduce a Configuration or Strategy Layer** for pricing rules so that the service can plug in different algorithms without touching the code.  
- **Add Unit Tests** for each method, especially edge cases (null inputs, extreme quantities).  
- **Document the Expected Behavior** of each method (e.g., does `calculatePriceQuantity` round to 2 decimal places?).  

Overall, the service layer is well‑structured but incomplete in critical areas. Completing the TODOs and tightening error handling will make it robust and production‑ready.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.pricing;

import java.math.BigDecimal;
import java.util.List;
import java.util.Locale;

import javax.inject.Inject;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.utils.ProductPriceUtils;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.catalog.product.availability.ProductAvailability;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.currency.Currency;

/**
 * Contains all the logic required to calculate product price
 * @author Carl Samson
 *
 */
@Service("pricingService")
public class PricingServiceImpl implements PricingService {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(PricingServiceImpl.class);
	

	@Inject
	private ProductPriceUtils priceUtil;
	
	@Override
	public FinalPrice calculateProductPrice(Product product) throws ServiceException {
		return priceUtil.getFinalPrice(product);
	}
	
	@Override
	public FinalPrice calculateProductPrice(Product product, Customer customer) throws ServiceException {
		/** TODO add rules for price calculation **/
		return priceUtil.getFinalPrice(product);
	}
	
	@Override
	public FinalPrice calculateProductPrice(Product product, List<ProductAttribute> attributes) throws ServiceException {
		return priceUtil.getFinalPrice(product, attributes);
	}
	
	@Override
	public FinalPrice calculateProductPrice(Product product, List<ProductAttribute> attributes, Customer customer) throws ServiceException {
		/** TODO add rules for price calculation **/
		return priceUtil.getFinalPrice(product, attributes);
	}
	
	@Override
	public BigDecimal calculatePriceQuantity(BigDecimal price, int quantity) {
		return price.multiply(new BigDecimal(quantity));
	}

	@Override
	public String getDisplayAmount(BigDecimal amount, MerchantStore store) throws ServiceException {
		try {
			return priceUtil.getStoreFormatedAmountWithCurrency(store,amount);
		} catch (Exception e) {
			LOGGER.error("An error occured when trying to format an amount " + amount.toString());
			throw new ServiceException(e);
		}
	}
	
	@Override
	public String getDisplayAmount(BigDecimal amount, Locale locale,
			Currency currency, MerchantStore store) throws ServiceException {
		try {
			return priceUtil.getFormatedAmountWithCurrency(locale, currency, amount);
		} catch (Exception e) {
			LOGGER.error("An error occured when trying to format an amunt " + amount.toString() + " using locale " + locale.toString() + " and currency " + currency.toString());
			throw new ServiceException(e);
		}
	}

	@Override
	public String getStringAmount(BigDecimal amount, MerchantStore store)
			throws ServiceException {
		try {
			return priceUtil.getAdminFormatedAmount(store, amount);
		} catch (Exception e) {
			LOGGER.error("An error occured when trying to format an amount " + amount.toString());
			throw new ServiceException(e);
		}
	}

	@Override
	public BigDecimal getAmount(String amount) throws ServiceException {

		try {
			return priceUtil.getAmount(amount);
		} catch (Exception e) {
			LOGGER.error("An error occured when trying to format an amount " + amount);
			throw new ServiceException(e);
		}

	}

	@Override
	public FinalPrice calculateProductPrice(ProductAvailability availability) throws ServiceException {

		return priceUtil.getFinalPrice(availability);
	}

	@Override
	public FinalPrice calculateProductPrice(ProductVariant variant) throws ServiceException {
		// TODO Auto-generated method stub
		return null;
	}


	
}



```
