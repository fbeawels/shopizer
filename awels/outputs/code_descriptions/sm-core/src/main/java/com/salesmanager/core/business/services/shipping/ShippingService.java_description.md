# ShippingService.java

## Review

## 1. Summary

The `ShippingService` interface defines the contract for all shipping‑related operations in the SalesManager e‑commerce platform.  
Its responsibilities include:

| Area | What it does |
|------|--------------|
| **Configuration** | Store shipping modules, supported countries, custom integration settings and overall shipping configuration. |
| **Quote Generation** | Compute shipping quotes for a shopping cart, optionally leveraging external gateway modules. |
| **Meta‑Data & Summary** | Provide meta‑information (e.g., tax rules) and build a shipping summary used for order totals. |
| **Utility** | Helpers such as package detail calculation and determining whether shipping is required. |

Key components referenced in the interface:

* **`MerchantStore`** – the current store / tenant.  
* **`IntegrationModule` / `IntegrationConfiguration`** – describe the external shipping gateways.  
* **`ShippingConfiguration` / `CustomIntegrationConfiguration`** – the merchant’s shipping rules.  
* **`ShippingQuote` / `ShippingOption` / `ShippingSummary`** – the result of a quote request.  
* **`Delivery`** – the customer's delivery address.  
* **`ShippingProduct` / `PackageDetails`** – the products to be shipped and the dimensional data needed to compute rates.  

The design follows a typical *service layer* pattern, exposing only business‑logic operations to callers. It relies on domain objects (e.g., `Country`, `Language`) from the same project and custom exception handling via `ServiceException`.

---

## 2. Detailed Description

### Core Flow

1. **Setup / Configuration**  
   * Merchant stores shipping preferences (national/international, shipping modules, tax rules).  
   * Configuration objects are persisted via the implementing class (likely a DAO or integration service).

2. **Quote Request**  
   * `getShippingQuote(...)` receives the cart ID, store, delivery details, a list of `ShippingProduct` (which contain product SKU, quantity, weight, dimensions, etc.), and a language.  
   * The implementation typically:
     * Retrieves the cart (via a separate `CartService`), merges with the passed `products` list if needed.  
     * Calls `getPackagesDetails(...)` to transform the product list into a list of `PackageDetails`.  
     * Determines which shipping modules are applicable by inspecting `getShippingModulesConfigured(...)` or `getSupportedCountries(...)`.  
     * Invokes each module’s API, collects `ShippingOption` objects, and aggregates them into a `ShippingQuote`.  

3. **User Selection & Summary**  
   * After the customer selects a shipping option, `getShippingSummary(...)` calculates the final shipping cost, tax (if `hasTaxOnShipping` is true), and any handling fees, returning a `ShippingSummary` that will be persisted with the order.

4. **Cleanup / Removal**  
   * The interface offers `removeShippingQuoteModuleConfiguration(...)` and `removeCustomShippingQuoteModuleConfiguration(...)` for housekeeping or migration.

### Assumptions & Constraints

* The interface assumes a *single tenant* per `MerchantStore` and that every method can access the underlying data store.  
* All methods throw `ServiceException`, implying that callers must handle or propagate this checked exception.  
* The domain model (`ShippingProduct`, `PackageDetails`, etc.) is tightly coupled to the service; if new product attributes are needed (e.g., hazardous material flag), the service signature may need extension.  
* International shipping logic (e.g., customs duties) is not represented; it may be handled elsewhere or via additional configuration.

### Architecture & Design Choices

* **Service Layer** – Separates business rules from controllers or data access.  
* **Domain‑Driven Design** – Uses rich domain objects (e.g., `ShippingMetaData`) instead of primitive DTOs.  
* **Extensibility** – Shipping modules are represented generically, allowing the addition of new providers without code changes.  
* **Error Handling** – Uniform `ServiceException` simplifies error propagation but may obscure specific failure modes (e.g., gateway timeout vs. validation error).

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `List<String> getSupportedCountries(MerchantStore)` | Return country codes the merchant allows shipping to. | `store` | List of ISO codes | None |
| `void setSupportedCountries(MerchantStore, List<String>)` | Persist allowed country codes. | `store`, `countryCodes` | None | Persists config |
| `List<IntegrationModule> getShippingMethods(MerchantStore)` | Fetch all available shipping modules for the store. | `store` | List of modules | None |
| `Map<String, IntegrationConfiguration> getShippingModulesConfigured(MerchantStore)` | Retrieve active module configurations keyed by module code. | `store` | Map | None |
| `void saveShippingQuoteModuleConfiguration(IntegrationConfiguration, MerchantStore)` | Persist a module config. | `configuration`, `store` | None | Persists config |
| `ShippingConfiguration getShippingConfiguration(MerchantStore)` | Retrieve the store’s shipping rules. | `store` | `ShippingConfiguration` | None |
| `void saveShippingConfiguration(ShippingConfiguration, MerchantStore)` | Persist shipping rules. | `shippingConfiguration`, `store` | None | Persists config |
| `void removeShippingQuoteModuleConfiguration(String, MerchantStore)` | Delete a module config. | `moduleCode`, `store` | None | Persists removal |
| `List<PackageDetails> getPackagesDetails(List<ShippingProduct>, MerchantStore)` | Convert product list to dimensional boxes. | `products`, `store` | List of `PackageDetails` | None |
| `ShippingQuote getShippingQuote(Long, MerchantStore, Delivery, List<ShippingProduct>, Language)` | Compute shipping options for a cart. | `shoppingCartId`, `store`, `delivery`, `products`, `language` | `ShippingQuote` | Might call external gateways |
| `IntegrationConfiguration getShippingConfiguration(String, MerchantStore)` | Get a specific module config. | `moduleCode`, `store` | `IntegrationConfiguration` | None |
| `CustomIntegrationConfiguration getCustomShippingConfiguration(String, MerchantStore)` | Get custom config for a module. | `moduleCode`, `store` | `CustomIntegrationConfiguration` | None |
| `void saveCustomShippingConfiguration(String, CustomIntegrationConfiguration, MerchantStore)` | Persist custom config. | `moduleCode`, `config`, `store` | None | Persists config |
| `void removeCustomShippingQuoteModuleConfiguration(String, MerchantStore)` | Delete a custom config. | `moduleCode`, `store` | None | Persists removal |
| `ShippingSummary getShippingSummary(MerchantStore, ShippingQuote, ShippingOption)` | Build final summary after user selection. | `store`, `shippingQuote`, `selectedShippingOption` | `ShippingSummary` | None |
| `List<Country> getShipToCountryList(MerchantStore, Language)` | Return a user‑friendly list of countries. | `store`, `language` | List of `Country` | None |
| `boolean requiresShipping(List<ShoppingCartItem>, MerchantStore)` | Determine if any item requires shipping. | `items`, `store` | `true/false` | None |
| `ShippingMetaData getShippingMetaData(MerchantStore)` | Return meta‑information about shipping rules. | `store` | `ShippingMetaData` | None |
| `boolean hasTaxOnShipping(MerchantStore)` | Whether shipping costs are taxed. | `store` | `true/false` | None |

**Reusable / Utility Methods**  
The interface itself does not provide utility methods; however, implementations often use internal helper methods (e.g., `buildPackageList`, `applyHandlingFees`) which can be extracted into a separate `ShippingUtils` class for easier unit testing.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.*` | Domain model, exceptions | Entirely internal to the project. |
| `ServiceException` | Custom checked exception | Wraps all failures; callers need to catch or re‑throw. |
| `IntegrationModule`, `IntegrationConfiguration` | Shipping gateway abstraction | Allows adding new modules without changing the interface. |
| `CustomIntegrationConfiguration` | Vendor‑specific settings | Provides flexibility for modules that need extra parameters. |
| `Country`, `Language` | Internationalisation | Leverages the existing i18n system. |

All dependencies are **project‑specific** (no third‑party libraries). This keeps the interface portable across the different layers of the SalesManager application but also couples it tightly to the domain model.

---

## 5. Additional Notes & Recommendations

### 1. Exception Strategy
* **Checked vs. Unchecked** – `ServiceException` is checked. Consider moving to an unchecked runtime exception or creating a hierarchy (`ConfigurationException`, `GatewayException`, etc.) for finer error handling.
* **Error Codes** – Add an error code or type to `ServiceException` so callers can differentiate issues programmatically.

### 2. Naming & Clarity
* **`getShippingConfiguration`** is overloaded: one accepts a store, another a module code + store. Consider renaming the latter to `getModuleConfiguration` to avoid confusion.
* **`getShippingModulesConfigured`** – the word *Configured* is redundant; `getConfiguredShippingModules` reads better.
* **`requiresShipping`** – The method name is fine, but it accepts a list of `ShoppingCartItem`. In many cases only a single item matters; consider passing the entire cart or an abstraction like `Cart`.

### 3. Parameter Validation
The interface itself cannot enforce non‑null arguments, but implementations should validate:
* `null` checks on all `MerchantStore` / `Delivery` parameters.
* Ensure `products` and `items` lists are not empty.

### 4. Extensibility
* **Multi‑Currency** – None of the methods accept a `Currency` parameter. If shipping rates differ by currency, the service layer should expose that.
* **Async Quote Retrieval** – Shipping gateways often involve network calls; exposing asynchronous methods (`CompletableFuture<ShippingQuote>`) could improve performance.
* **Event Hooks** – Allow callers to subscribe to events such as *quoteGenerated* or *moduleConfigured*.

### 5. Documentation
* Several methods lack detailed Javadoc (e.g., `saveShippingConfiguration`, `removeCustomShippingQuoteModuleConfiguration`). Adding descriptions of expected input formats and side‑effects would help implementers and callers.
* Include examples in comments for complex operations such as `getShippingQuote`.

### 6. Unit Testing & Mocking
* Because the interface abstracts all external dependencies, it is straightforward to mock in unit tests. However, documenting expected behavior for each method (e.g., what happens when a module fails) will improve test coverage.

### 7. Performance Considerations
* Methods that call external gateways (`getShippingQuote`) may become bottlenecks. Implementations should cache rates per destination/weight for a configurable period or support batch requests.
* `getSupportedCountries` and `getShipToCountryList` might involve database lookups; consider using in‑memory caches for frequently accessed data.

### 8. Future Enhancements
* **Dynamic Package Sizing** – Allow merchants to configure automatic box sizing rules or provide a plug‑in for custom logic.
* **Custom Shipping Rules** – Introduce a rule engine to allow complex conditions (e.g., free shipping over X amount).
* **Integration with Analytics** – Expose metrics such as number of quotes per module, average cost, etc.

---

### Bottom Line

The `ShippingService` interface provides a clear, well‑structured contract for all shipping‑related functionality. Its focus on domain objects and extensibility makes it suitable for a modular e‑commerce platform. With minor refinements in naming, exception handling, and documentation, the interface will be even more robust and developer‑friendly.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.shipping;

import java.util.List;
import java.util.Map;


import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingMetaData;
import com.salesmanager.core.model.shipping.ShippingOption;
import com.salesmanager.core.model.shipping.ShippingProduct;
import com.salesmanager.core.model.shipping.ShippingQuote;
import com.salesmanager.core.model.shipping.ShippingSummary;
import com.salesmanager.core.model.shoppingcart.ShoppingCartItem;
import com.salesmanager.core.model.system.CustomIntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationConfiguration;
import com.salesmanager.core.model.system.IntegrationModule;



public interface ShippingService {

	/**
	 * Returns a list of supported countries (ship to country list) configured by merchant
	 * when the merchant configured shipping National and has saved a list of ship to country
	 * from the list
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	List<String> getSupportedCountries(MerchantStore store)
			throws ServiceException;

	void setSupportedCountries(MerchantStore store,
			List<String> countryCodes) throws ServiceException;

	/**
	 * Returns a list of available shipping modules
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	List<IntegrationModule> getShippingMethods(MerchantStore store)
			throws ServiceException;

	
	/**
	 * Returns a list of configured shipping modules for a given merchant
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	Map<String, IntegrationConfiguration> getShippingModulesConfigured(
			MerchantStore store) throws ServiceException;

	/**
	 * Adds a Shipping configuration
	 * @param configuration
	 * @param store
	 * @throws ServiceException
	 */
	void saveShippingQuoteModuleConfiguration(IntegrationConfiguration configuration,
			MerchantStore store) throws ServiceException;

	/**
	 * ShippingType (NATIONAL, INTERNATIONSL)
	 * ShippingBasisType (SHIPPING, BILLING)
	 * ShippingPriceOptionType (ALL, LEAST, HIGHEST)
	 * Packages
	 * Handling
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	ShippingConfiguration getShippingConfiguration(MerchantStore store)
			throws ServiceException;

	/**
	 * Saves ShippingConfiguration for a given MerchantStore
	 * @param shippingConfiguration
	 * @param store
	 * @throws ServiceException
	 */
	void saveShippingConfiguration(ShippingConfiguration shippingConfiguration,
			MerchantStore store) throws ServiceException;

	void removeShippingQuoteModuleConfiguration(String moduleCode,
			MerchantStore store) throws ServiceException;

	/**
	 * Provides detailed information on boxes that will be used
	 * when getting a ShippingQuote
	 * @param products
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	List<PackageDetails> getPackagesDetails(List<ShippingProduct> products,
			MerchantStore store) throws ServiceException;

	/**
	 * Get a list of ShippingQuote from a configured
	 * shipping gateway. The quotes are displayed to the end user so he can pick
	 * the ShippingOption he wants
	 * @param store
	 * @param shoppingCartId
	 * @param customer
	 * @param products
	 * @param language
	 * @return
	 * @throws ServiceException
	 */
	ShippingQuote getShippingQuote(Long shoppingCartId, MerchantStore store, Delivery delivery,
			List<ShippingProduct> products, Language language)
			throws ServiceException;
	

	/**
	 * Returns a shipping module configuration given a moduleCode
	 * @param moduleCode
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	IntegrationConfiguration getShippingConfiguration(String moduleCode,
			MerchantStore store) throws ServiceException;
	
	/**
	 * Retrieves the custom configuration for a given module
	 * @param moduleCode
	 * @param store
	 * @return
	 * @throws ServiceException
	 */


	CustomIntegrationConfiguration getCustomShippingConfiguration(
			String moduleCode, MerchantStore store) throws ServiceException;

	/**
	 * Weight based configuration
	 * @param moduleCode
	 * @param shippingConfiguration
	 * @param store
	 * @throws ServiceException
	 */
	void saveCustomShippingConfiguration(String moduleCode,
			CustomIntegrationConfiguration shippingConfiguration,
			MerchantStore store) throws ServiceException;

	/**
	 * Removes a custom shipping quote
	 * module
	 * @param moduleCode
	 * @param store
	 * @throws ServiceException
	 */
	void removeCustomShippingQuoteModuleConfiguration(String moduleCode,
			MerchantStore store) throws ServiceException;

	/**
	 * The {@link ShippingSummary} is built from the ShippingOption the user has selected
	 * The ShippingSummary is used for order calculation
	 * @param store
	 * @param shippingQuote
	 * @param selectedShippingOption
	 * @return
	 * @throws ServiceException
	 */
	ShippingSummary getShippingSummary(MerchantStore store, ShippingQuote shippingQuote, 
			ShippingOption selectedShippingOption) throws ServiceException;

	/**
	 * Returns a list of supported countries (ship to country list) configured by merchant
	 * If the merchant configured shipping National, then only store country will be in the list
	 * If the merchant configured shipping International, then the list of accepted country is returned
	 * from the list
	 * @param store
	 * @param language
	 * @return
	 * @throws ServiceException
	 */
	List<Country> getShipToCountryList(MerchantStore store, Language language)
			throws ServiceException;
	
	/**
	 * Determines if Shipping should be proposed to the customer
	 * @param items
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	boolean requiresShipping(List<ShoppingCartItem> items, MerchantStore store) throws ServiceException;

	/**
	 * Returns shipping metadata and how shipping is configured for a given store
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	ShippingMetaData getShippingMetaData(MerchantStore store) throws ServiceException;
	
	/**
	 * Based on merchant configurations will return if tax must be calculated on shipping
	 * @param store
	 * @return
	 * @throws ServiceException
	 */
	boolean hasTaxOnShipping(MerchantStore store) throws ServiceException;


}


```
