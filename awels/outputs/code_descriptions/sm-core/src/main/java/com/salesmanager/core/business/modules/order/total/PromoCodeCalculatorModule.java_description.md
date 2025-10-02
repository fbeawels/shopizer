# PromoCodeCalculatorModule.java

## Review

## 1. Summary
The `PromoCodeCalculatorModule` is a Spring‑managed component that implements the `OrderTotalPostProcessorModule` interface.  
Its responsibility is to apply a promotional coupon (if one is present on an `OrderSummary`) by evaluating a Drools rule set (`PromoCoupon.drl`) and, if a discount is produced, creating an `OrderTotal` entry that represents the negative amount to be subtracted from the order’s subtotal.

Key components  
| Component | Role |
|-----------|------|
| `DroolsBeanFactory` | Provides a `KieSession` for executing the Drools rules. |
| `PricingService` | Calculates the final price of a product, used to compute the discount amount. |
| `PromoCoupon.drl` | Drools rule file that determines whether a coupon is valid and what discount should be applied. |
| `OrderTotalResponse`, `OrderTotalInputParameters` | DTOs used by the rule engine – the former is a global that receives the result, the latter holds input data such as the promo code and current date. |

The code is a classic example of *data‑driven rule processing* (Drools) combined with a service‑layer approach for business logic. It follows the **Strategy** pattern – each `OrderTotalPostProcessorModule` encapsulates a different calculation strategy that can be plugged into the order‑total pipeline.

---

## 2. Detailed Description
### Initialization
* Spring injects `DroolsBeanFactory` and `PricingService` via `@Autowired`.
* The module’s `name` and `code` fields can be set externally (e.g., via XML or a component scan).

### Execution Flow (`caculateProductPiceVariation`)
1. **Pre‑validation**  
   * `summary` and `store` are checked for `null`.  
   * If the order contains no promo code (`summary.getPromoCode()` is blank), the method returns `null` immediately.

2. **Drools Session Setup**  
   * A new `KieSession` is created from `PromoCoupon.drl`.  
   * An `OrderTotalInputParameters` instance is populated with the promo code and the current date.  
   * A fresh `OrderTotalResponse` is created and registered as a global named `"total"`.  
   * The input parameters are inserted and `fireAllRules()` is called.

3. **Result Handling**  
   * If the response contains a discount, a new `OrderTotal` is built:  
     * The total type is `SUBTOTAL`, title and code are hard‑coded constants.  
     * The discount is calculated as `productPrice * discountPercentage * quantity`.  
   * The negative value is set as the `OrderTotal.value`.  
   * The resulting `OrderTotal` is returned; otherwise, `null` is returned.

4. **Cleanup**  
   * The `KieSession` is **not** explicitly disposed – a potential resource leak on high‑traffic systems.

### Assumptions & Constraints
* The discount percentage (`resp.getDiscount()`) is assumed to be a *fraction* (e.g., `0.10` for 10 %).
* The product price calculation via `PricingService` is deterministic and side‑effect free.
* The rules file `PromoCoupon.drl` lives in the classpath under `com/salesmanager/drools/rules/`.
* No explicit handling of coupon expiration or usage limits – the rule file must provide these checks.

### Architecture & Design Choices
* **Rule‑Based vs Code‑Based**: Using Drools allows non‑developers to add or modify coupon logic without redeploying Java code.
* **Global & Insert**: The rule engine communicates via globals and inserted facts, a common pattern in Drools.
* **Single Responsibility**: The method’s primary job is to orchestrate the rule execution; the actual discount calculation remains in the `PricingService`.

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getName()` | Returns the module’s name. | None | `String` | None |
| `setName(String)` | Sets the module’s name. | `String` | void | Sets field |
| `getCode()` | Returns the module’s code. | None | `String` | None |
| `setCode(String)` | Sets the module’s code. | `String` | void | Sets field |
| `caculateProductPiceVariation(...)` | Calculates a discount `OrderTotal` based on a promo code. | `OrderSummary`, `ShoppingCartItem`, `Product`, `Customer`, `MerchantStore` | `OrderTotal` or `null` | Inserts facts into Drools, creates an `OrderTotal` if a discount exists. |

**Reusable/Utility**  
* The method relies on `Validate` from Apache Commons Lang for null checks – a lightweight utility.

---

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.lang3` | Third‑party | Used for `StringUtils` and `Validate`. |
| `org.kie.api` & `org.kie.internal` | Third‑party | Drools rule engine API. |
| `org.springframework` | Framework | Spring DI (`@Component`, `@Autowired`). |
| `com.salesmanager.core.*` | Project | Domain models, constants, services, and the module interface. |

No platform‑specific dependencies; all code is plain Java with standard libraries.

---

## 5. Additional Notes

### Strengths
* Clear separation between rule definition (Drools) and Java orchestration.  
* Uses Spring for lifecycle and dependency injection, simplifying configuration.  
* Straightforward flow: validate, execute rules, interpret results.

### Weaknesses / Edge Cases
1. **Resource Leak** – The `KieSession` is never disposed. In a high‑traffic environment this will exhaust native resources.
2. **Null‑Pointer Risks** – If `product` or `shoppingCartItem` are null, the method will throw a `NullPointerException` later. No validation for these parameters.
3. **Precision Issues** – `resp.getDiscount()` returns a `Double`; multiplying `BigDecimal` by a `Double` requires conversion to `BigDecimal`, which can introduce rounding errors.  
   * Recommended: store discount as `BigDecimal` or as an integer percentage (e.g., 10 for 10 %).
4. **Hard‑coded Constants** – The `OrderTotal` code, title, and type are hard‑coded; this limits flexibility if other modules need similar discounts but with different identifiers.
5. **Missing Expiration Check** – The comment “TODO check expiration” indicates incomplete logic. Depending on the rule file, this might be redundant, but the placeholder suggests a future enhancement.
6. **Method Name Typo** – `caculateProductPiceVariation` is misspelled. Should be `calculateProductPriceVariation`.
7. **Error Handling** – The method declares `throws Exception` but never throws it. A more specific exception hierarchy or a custom checked exception would be clearer.

### Potential Enhancements
* **Dispose KieSession**  
  ```java
  try (KieSession kieSession = droolsBeanFactory.getKieSession(...)) {
      // rule execution
  }
  ```
* **Parameter Validation** – Add `Validate.notNull(product)` and `Validate.notNull(shoppingCartItem)`.
* **Decimal Safety** – Convert `resp.getDiscount()` to `BigDecimal` before multiplying:
  ```java
  BigDecimal discount = BigDecimal.valueOf(resp.getDiscount()).setScale(4, RoundingMode.HALF_UP);
  ```
* **Refactor Constants** – Expose discount-related constants via a configuration bean or enum.
* **Unit Tests** – Provide tests that mock the `DroolsBeanFactory` and `PricingService` to verify correct discount calculation under different rule outcomes.
* **Logging** – Add informative log statements (e.g., “Promo code applied: 10% off”).
* **Internationalization** – The `OrderTotal.title` uses a constant; consider retrieving a localized string from a message source.

### Summary
The module implements a solid, rule‑based approach to applying promo‑code discounts. It is functional but would benefit from small but important fixes—resource cleanup, type safety, parameter validation, and clearer naming—to make it robust, maintainable, and ready for production usage.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.order.total;

import java.math.BigDecimal;
import java.util.Date;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.kie.api.runtime.KieSession;
import org.kie.internal.io.ResourceFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.salesmanager.core.business.configuration.DroolsBeanFactory;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.services.catalog.pricing.PricingService;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.price.FinalPrice;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.OrderSummary;
import com.salesmanager.core.model.order.OrderTotal;
import com.salesmanager.core.model.order.OrderTotalType;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.modules.order.total.OrderTotalPostProcessorModule;

@Component
public class PromoCodeCalculatorModule implements OrderTotalPostProcessorModule {
	
	
	@Autowired
	private DroolsBeanFactory droolsBeanFactory;
	
	@Autowired
	private PricingService pricingService;

	private String name;
	private String code;

	@Override
	public String getName() {
		// TODO Auto-generated method stub
		return name;
	}

	@Override
	public void setName(String name) {
		this.name = name;

	}

	@Override
	public String getCode() {
		// TODO Auto-generated method stub
		return code;
	}

	@Override
	public void setCode(String code) {
		this.code = code;
	}

	@Override
	public OrderTotal caculateProductPiceVariation(OrderSummary summary, ShoppingCartItem shoppingCartItem,
			Product product, Customer customer, MerchantStore store) throws Exception {
		
		Validate.notNull(summary, "OrderTotalSummary must not be null");
		Validate.notNull(store, "MerchantStore must not be null");
		
		if(StringUtils.isBlank(summary.getPromoCode())) {
			return null;
		}
		
		KieSession kieSession=droolsBeanFactory.getKieSession(ResourceFactory.newClassPathResource("com/salesmanager/drools/rules/PromoCoupon.drl"));
		
		OrderTotalResponse resp = new OrderTotalResponse();
		
		OrderTotalInputParameters inputParameters = new OrderTotalInputParameters();
		inputParameters.setPromoCode(summary.getPromoCode());
		inputParameters.setDate(new Date());
		
        kieSession.insert(inputParameters);
        kieSession.setGlobal("total",resp);
        kieSession.fireAllRules();

		if(resp.getDiscount() != null) {
			
			OrderTotal orderTotal = null;
			if(resp.getDiscount() != null) {
					orderTotal = new OrderTotal();
					orderTotal.setOrderTotalCode(Constants.OT_DISCOUNT_TITLE);
					orderTotal.setOrderTotalType(OrderTotalType.SUBTOTAL);
					orderTotal.setTitle(Constants.OT_SUBTOTAL_MODULE_CODE);
					orderTotal.setText(summary.getPromoCode());
					
					//calculate discount that will be added as a negative value
					FinalPrice productPrice = pricingService.calculateProductPrice(product);
					
					Double discount = resp.getDiscount();
					BigDecimal reduction = productPrice.getFinalPrice().multiply(new BigDecimal(discount));
					reduction = reduction.multiply(new BigDecimal(shoppingCartItem.getQuantity()));
					
					orderTotal.setValue(reduction);//discount value
					
					//TODO check expiration
			}
				
			
			
			return orderTotal;
			
		}
		
		
		
		return null;
	}

}



```
