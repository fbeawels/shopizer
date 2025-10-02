# ManufacturerShippingCodeOrderTotalModuleImpl.java

## Review

## 1. Summary
**Purpose**  
`ManufacturerShippingCodeOrderTotalModuleImpl` is a Spring‐managed component that implements the `OrderTotalPostProcessorModule` interface.  
Its role is to apply a *price variation* (typically a discount) to a product in a shopping cart based on the product’s manufacturer and the shipping method chosen by the customer. The variation is calculated by applying a discount factor retrieved from a decision‑making engine (originally a Drools `StatelessKnowledgeSession`, now commented out) to the final price of the product.

**Key components**

| Component | Role |
|-----------|------|
| `PricingService` | Computes the final price of a product (`FinalPrice`) |
| `OrderTotalInputParameters` | Holds the context needed for the decision engine |
| `OrderTotal` | Encapsulates the calculated adjustment (discount) that will be added to the order summary |
| `OrderTotalPostProcessorModule` | Interface that this class implements, allowing it to plug into the order total calculation pipeline |

**Notable design patterns / frameworks**

- Spring’s `@Component` for dependency injection.
- Apache Commons `Validate` for pre‑condition checks.
- (Commented out) Drools (rule engine) – indicates an intention to separate business rules from code.
- Use of constants (`Constants.OT_DISCOUNT_TITLE`, `Constants.OT_SUBTOTAL_MODULE_CODE`) for standard titles/codes.

---

## 2. Detailed Description

### Execution Flow
1. **Validation** – The method `caculateProductPiceVariation` first verifies that the `product` and its `manufacturer` are not null.  
2. **Shipping check** – If the order summary does not contain a shipping summary (`summary.getShippingSummary() == null`), the method immediately returns `null` (no variation applied).  
3. **Input preparation** – An `OrderTotalInputParameters` instance is created and populated with the manufacturer code and shipping option code.  
4. **Decision engine (currently disabled)** – The commented code shows that a Drools session would normally be created, the parameters inserted, and rules fired to compute a discount. In the current state, no rules are executed, so `inputParameters.getDiscount()` will always be `null`.  
5. **Discount application** – If a discount value is present, the method:
   - Retrieves the final price of the product via `pricingService.calculateProductPrice(product)`.
   - Multiplies that price by the discount factor and the quantity in the cart to obtain the total reduction.
   - Constructs an `OrderTotal` object, setting its code, type, title, and value.
6. **Return** – The `OrderTotal` (or `null` if no discount) is returned to the caller.

### Assumptions & Constraints
- The decision engine (rules) is expected to populate `inputParameters.discount`. The current implementation assumes this field will be set externally; otherwise, no adjustment is made.
- `pricingService` must be correctly injected and available; no null‑check is performed on this field.
- The calculation treats the discount as a *percentage* expressed as a decimal (e.g., `0.10` for 10%). The code does not enforce that the value lies within a realistic range.
- The `OrderTotal` is created with type `SUBTOTAL` even though it represents a discount. The naming of constants (`OT_DISCOUNT_TITLE`, `OT_SUBTOTAL_MODULE_CODE`) suggests a mismatch that could be clarified.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `caculateProductPiceVariation(OrderSummary summary, ShoppingCartItem shoppingCartItem, Product product, Customer customer, MerchantStore store)` | Computes a discount (price variation) based on manufacturer and shipping method. | `summary`, `shoppingCartItem`, `product`, `customer`, `store` | `OrderTotal` or `null` | None (pure calculation, except logging). |
| `getPricingService()` | Getter for injected `PricingService`. | – | `PricingService` | – |
| `setPricingService(PricingService pricingService)` | Setter for `PricingService`. | `pricingService` | – | Sets internal reference. |
| `getName()`, `setName(String name)` | Get/set module name. | – / `name` | – / – | – |
| `getCode()`, `setCode(String code)` | Get/set module code. | – / `code` | – / – | – |

### Utility / Reusable Methods
The class itself contains no standalone utility methods beyond the standard getters/setters. The core logic is encapsulated in `caculateProductPiceVariation`.

---

## 4. Dependencies

| External Lib | Type | Purpose |
|--------------|------|---------|
| `org.apache.commons.lang3.Validate` | Third‑party | Simple null/argument validation. |
| `org.slf4j.Logger` / `LoggerFactory` | Third‑party | Logging. |
| Spring (`@Component`) | Framework | Dependency injection and bean lifecycle. |
| `com.salesmanager.core.*` | Internal | Domain models (`Product`, `OrderSummary`, `OrderTotal`, etc.) and services (`PricingService`). |
| (Commented) Drools (`StatelessKnowledgeSession`, `KnowledgeBase`, `KieContainer`) | Third‑party | Business rule engine (currently unused). |

No platform‑specific dependencies; everything is pure Java.

---

## 5. Additional Notes & Recommendations

### 1. Correct Method Naming
`caculateProductPiceVariation` contains typographical errors (`caculate` → `calculate`, `Pice` → `Price`). Renaming improves readability and consistency with the interface.

### 2. Decision Engine Activation
If the rule engine is no longer required, remove the commented code and the dependency on Drools. If it is required, re‑enable the session creation, insertion, and rule firing logic. Ensure that the rules set the discount correctly.

### 3. PricingService Injection Check
Add a `Validate.notNull(pricingService, "PricingService must be injected")` at the beginning of `caculateProductPiceVariation` to avoid a `NullPointerException`.

### 4. Discount Sign Handling
The current implementation multiplies the final price by the discount factor. If the discount should be *negative* (i.e., reduce the total), ensure that `inputParameters.getDiscount()` is a negative value or explicitly negate the result:
```java
BigDecimal reduction = productPrice.getFinalPrice()
                                   .multiply(BigDecimal.valueOf(discount))
                                   .multiply(BigDecimal.valueOf(shoppingCartItem.getQuantity()));
orderTotal.setValue(reduction.negate()); // if you want a negative number
```

### 5. Validation of Discount Range
Add checks to confirm that the discount is between `0` and `1` (or whatever business rule applies). Throw an exception or log a warning otherwise.

### 6. Use of `customer` and `store`
These parameters are currently unused. Either remove them from the method signature or integrate them into the decision logic if they are relevant.

### 7. OrderTotal Code & Type
The `OrderTotal` created uses:
```java
orderTotal.setOrderTotalCode(Constants.OT_DISCOUNT_TITLE);
orderTotal.setOrderTotalType(OrderTotalType.SUBTOTAL);
orderTotal.setTitle(Constants.OT_SUBTOTAL_MODULE_CODE);
```
The mix of a discount code with a subtotal type is confusing. Clarify the intended semantics:
- If the adjustment is a discount, consider `OrderTotalType.DISCOUNT`.
- Ensure the title reflects the nature of the adjustment.

### 8. Logging
Add logs for cases where `discount == null` or when a `null` shipping summary is encountered. This will aid in debugging.

### 9. Exception Handling
The method declares `throws Exception`. Replace it with more specific checked exceptions (e.g., `InvalidOrderException`) or handle exceptions internally to keep the interface clean.

### 10. Documentation & Javadoc
Provide method‑level Javadoc that explains the algorithm, input assumptions, and the expected format of the discount value.

### 11. Unit Tests
Create unit tests covering:
- Normal discount calculation.
- Null shipping summary → returns `null`.
- Null product/manufacturer → throws `IllegalArgumentException`.
- Discount outside expected range → handled gracefully.

--- 

Overall, the component is straightforward but currently incomplete (missing rule execution). Addressing the issues above will make the module robust, maintainable, and easier to integrate into the larger order‑processing pipeline.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.order.total;

import java.math.BigDecimal;

import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

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


/**
 * Add variation to the OrderTotal
 * This has to be defined in shopizer-core-ordertotal-processors
 * @author carlsamson
 *
 */
@Component
public class ManufacturerShippingCodeOrderTotalModuleImpl implements OrderTotalPostProcessorModule {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ManufacturerShippingCodeOrderTotalModuleImpl.class);
	
	private String name;
	private String code;
	
	//private StatelessKnowledgeSession orderTotalMethodDecision;//injected from xml file
	
	//private KnowledgeBase kbase;//injected from xml file
	
	//@Inject
	//KieContainer kieManufacturerBasedPricingContainer;
	

	PricingService pricingService;
	

	
	public PricingService getPricingService() {
		return pricingService;
	}

	public void setPricingService(PricingService pricingService) {
		this.pricingService = pricingService;
	}

	@Override
	public OrderTotal caculateProductPiceVariation(final OrderSummary summary, ShoppingCartItem shoppingCartItem, Product product, Customer customer, MerchantStore store)
			throws Exception {

		
	    Validate.notNull(product,"product must not be null");
		Validate.notNull(product.getManufacturer(),"product manufacturer must not be null");
		
		//requires shipping summary, otherwise return null
		if(summary.getShippingSummary()==null) {
			return null;
		}

		OrderTotalInputParameters inputParameters = new OrderTotalInputParameters();
		inputParameters.setItemManufacturerCode(product.getManufacturer().getCode());
		
		
		inputParameters.setShippingMethod(summary.getShippingSummary().getShippingOptionCode());
		
		LOGGER.debug("Setting input parameters " + inputParameters.toString());
		
/*        KieSession kieSession = kieManufacturerBasedPricingContainer.newKieSession();
        kieSession.insert(inputParameters);
        kieSession.fireAllRules();*/
		
		
		//orderTotalMethodDecision.execute(inputParameters);
		
		
		LOGGER.debug("Applied discount " + inputParameters.getDiscount());
		
		OrderTotal orderTotal = null;
		if(inputParameters.getDiscount() != null) {
				orderTotal = new OrderTotal();
				orderTotal.setOrderTotalCode(Constants.OT_DISCOUNT_TITLE);
				orderTotal.setOrderTotalType(OrderTotalType.SUBTOTAL);
				orderTotal.setTitle(Constants.OT_SUBTOTAL_MODULE_CODE);
				
				//calculate discount that will be added as a negative value
				FinalPrice productPrice = pricingService.calculateProductPrice(product);
				
				Double discount = inputParameters.getDiscount();
				BigDecimal reduction = productPrice.getFinalPrice().multiply(new BigDecimal(discount));
				reduction = reduction.multiply(new BigDecimal(shoppingCartItem.getQuantity()));
				
				orderTotal.setValue(reduction);
		}
			
		
		
		return orderTotal;


	}
	
/*	public KnowledgeBase getKbase() {
		return kbase;
	}


	public void setKbase(KnowledgeBase kbase) {
		this.kbase = kbase;
	}

	public StatelessKnowledgeSession getOrderTotalMethodDecision() {
		return orderTotalMethodDecision;
	}

	public void setOrderTotalMethodDecision(StatelessKnowledgeSession orderTotalMethodDecision) {
		this.orderTotalMethodDecision = orderTotalMethodDecision;
	}*/

	@Override
	public String getName() {
		return name;
	}

	@Override
	public void setName(String name) {
		this.name = name;
	}

	@Override
	public String getCode() {
		return code;
	}

	@Override
	public void setCode(String code) {
		this.code = code;
	}



}



```
