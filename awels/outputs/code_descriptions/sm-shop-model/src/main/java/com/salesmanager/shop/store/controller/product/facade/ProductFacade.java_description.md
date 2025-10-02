# ProductFacade.java

## Review

## 1. Summary
The **`ProductFacade`** interface defines a contract for a high‑level product service exposed to the presentation layer of the SalesManager shop application.  
It abstracts the underlying domain model (`Product`, `ProductCriteria`, etc.) and offers a read‑only view (`ReadableProduct`, `ReadableProductList`, `ReadableProductPrice`) that can be safely serialized and returned to clients.  

Key points:
- **Purpose** – Provide convenient read‑only product look‑ups, listings, and related‑product queries.
- **Components** – Methods to fetch by ID, code, SKU, friendly URL; filter by criteria; and retrieve related items.
- **Design patterns** – Typical *Facade* pattern: a single entry point to complex subsystems.  
- **Frameworks/Libraries** – Relies on the SalesManager core model classes; no external frameworks are directly referenced in the interface.

---

## 2. Detailed Description
The interface is intentionally lightweight; concrete implementations would inject services such as `ProductService`, `MerchantStoreService`, or repositories.  
The typical flow for a request:

1. **Client** (controller or other facade consumer) calls one of the `ProductFacade` methods, passing the current `MerchantStore`, a `Language` object, and any identifiers (ID, code, SKU, etc.).
2. **Facade implementation** validates inputs, retrieves the underlying `Product` entity via core services, converts it into a `ReadableProduct` (or list/price) using a mapper or DTO factory.
3. **Result** – A DTO that is safe to expose via REST or UI layers.

Cleanup is not a concern because the methods are read‑only and stateless; any required resource closing is handled by the underlying services or container-managed transactions.

Assumptions / Constraints:
- All methods throw generic `Exception`; callers should handle or propagate.
- The caller must provide a fully initialized `MerchantStore` and `Language` context; null checks are expected to be performed by the implementation.
- The methods are designed for *read* operations only; no mutating operations are exposed.

---

## 3. Functions/Methods
| Method | Signature | Purpose | Inputs | Outputs | Side Effects |
|--------|-----------|---------|--------|---------|--------------|
| `Product getProduct(Long id, MerchantStore store)` | Retrieve the *entity* `Product` by its primary key within a store. | Low‑level fetch used internally; exposes the domain object. | `id` – product identifier; `store` – merchant context. | `Product` entity or `null`. | None. |
| `ReadableProduct getProductByCode(MerchantStore store, String uniqueCode, Language language)` | Find a product by its unique code and convert to a DTO. | Exposes a read‑only view for UI. | `store`, `uniqueCode`, `language`. | `ReadableProduct` or `null`. | None. |
| `ReadableProduct getProduct(MerchantStore store, String sku, Language language)` | Look up a product by SKU. | Same as above but SKU based. | `store`, `sku`, `language`. | `ReadableProduct` or `null`. | Throws `Exception`. |
| `ReadableProduct getProductBySeUrl(MerchantStore store, String friendlyUrl, Language language)` | Retrieve product via its SEO friendly URL (slug). | Enables clean URLs. | `store`, `friendlyUrl`, `language`. | `ReadableProduct` or `null`. | Throws `Exception`. |
| `ReadableProductList getProductListsByCriterias(MerchantStore store, Language language, ProductCriteria criterias)` | Filter products by a set of criteria and return a paginated list. | Used for product search pages. | `store`, `language`, `criterias`. | `ReadableProductList` containing a collection of `ReadableProduct`. | Throws `Exception`. |
| `List<ReadableProduct> relatedItems(MerchantStore store, Product product, Language language)` | Fetch items related to a given product. | For “Related Products” carousel. | `store`, `product` entity, `language`. | List of `ReadableProduct`. | Throws `Exception`. |

**Reusable/Utility Methods**  
The interface itself does not provide utilities; however, any implementation is expected to leverage a mapper (e.g., `ProductMapper`) to convert `Product` entities to `ReadableProduct` DTOs, ensuring separation of concerns.

---

## 4. Dependencies
| Component | Type | Notes |
|-----------|------|-------|
| `com.salesmanager.core.model.catalog.product.Product` | Domain entity | Core model; no external dependency. |
| `ProductCriteria` | Value object | Used for filtering; part of core. |
| `MerchantStore` | Domain entity | Stores merchant data. |
| `Language` | Reference entity | Language configuration. |
| `ReadableProduct`, `ReadableProductList`, `ReadableProductPrice` | DTOs | Safe serializable representations. |
| Java Standard Library | Standard | Basic collections, exceptions. |

No external frameworks (Spring, Hibernate, etc.) are directly referenced in this interface, but typical implementations would depend on them.

---

## 5. Additional Notes
### Strengths
- **Clear separation** between domain objects (`Product`) and view objects (`ReadableProduct`), preventing leakage of internal state.
- **Facade pattern** simplifies client interactions and hides complexity.
- **Consistent method signatures** across different lookup strategies.

### Potential Issues / Edge Cases
- **Generic `Exception`** in signatures: forces callers to handle a broad exception set; consider more specific exceptions (`ProductNotFoundException`, `InvalidStoreException`, etc.).
- **Null handling**: The contract does not specify whether `null` is returned when no product is found or an exception is thrown. Clarify behavior in documentation.
- **Pagination**: `ReadableProductList` is likely paginated, but the method does not expose page parameters; the `ProductCriteria` may encapsulate them, but this should be documented.
- **Security**: Methods accept a `MerchantStore` parameter but do not enforce permission checks. Implementations should validate that the current user has access to the store.
- **Locale fallback**: If the requested language is not available, the behavior (fallback to default language or throw) should be defined.

### Future Enhancements
- **Caching**: Add read‑through or read‑write caching layers to reduce database hits for frequently accessed products.
- **Search/Filtering API**: Expose more advanced search features (e.g., full‑text, price range).
- **Bulk operations**: Methods to retrieve multiple products by a list of SKUs or IDs.
- **Event hooks**: Allow listeners to react to product retrievals for analytics or logging.

Overall, the interface provides a solid foundation for a product read service. Clarifying the contract details (exception types, null semantics, pagination) would strengthen its usability and robustness.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.product.facade;

import java.util.List;

import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.ProductCriteria;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.catalog.product.ProductPriceRequest;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.catalog.product.ReadableProductList;
import com.salesmanager.shop.model.catalog.product.ReadableProductPrice;

public interface ProductFacade {


  
  
  /**
   * 
   * @param id
   * @param store
   * @return
   */
  Product getProduct(Long id, MerchantStore store);

  /**
   * Reads a product by code
   *
   * @param store
   * @param uniqueCode
   * @param language
   * @return
   * @throws Exception
   */
  ReadableProduct getProductByCode(MerchantStore store, String uniqueCode, Language language);

  /**
   * Get a product by sku and store
   *
   * @param store
   * @param sku
   * @param language
   * @return
   * @throws Exception
   */
  ReadableProduct getProduct(MerchantStore store, String sku, Language language) throws Exception;

  /**
   * Get a Product by friendlyUrl (slug), store and language
   *
   * @param store
   * @param friendlyUrl
   * @param language
   * @return
   * @throws Exception
   */
  ReadableProduct getProductBySeUrl(MerchantStore store, String friendlyUrl, Language language) throws Exception;

  /**
   * Filters a list of product based on criteria
   *
   * @param store
   * @param language
   * @param criterias
   * @return
   * @throws Exception
   */
  ReadableProductList getProductListsByCriterias(MerchantStore store, Language language,
      ProductCriteria criterias) throws Exception;

  /**
   * Get related items
   *
   * @param store
   * @param product
   * @param language
   * @return
   * @throws Exception
   */
  List<ReadableProduct> relatedItems(MerchantStore store, Product product, Language language)
      throws Exception;
  
 
}



```
