# SearchService.java

## Review

## 1. Summary
The `SearchService` interface defines the contract for product‑search functionality within the Sales Manager application.  
Key responsibilities include:

| Responsibility | Method(s) | Purpose |
|----------------|-----------|---------|
| Indexing products (or variants) | `index` | Add or update product data in the search index. |
| Deleting products | `deleteDocument` | Remove a product from the search index. |
| Keyword auto‑complete | `searchKeywords` | Provide similar keyword suggestions based on partial user input. |
| Full product search | `search` | Execute a search query and return paginated results. |
| Document retrieval | `getDocument` | Fetch the raw indexed document by ID. |

The interface is framework‑agnostic; implementations can target any search backend (Elasticsearch, Solr, custom in‑memory index, etc.). It relies on domain objects (`Product`, `ProductVariant`, `MerchantStore`) and a generic request/response model (`SearchRequest`, `SearchResponse`, `Document`) defined in an external module `modules.commons.search`.

Design patterns at play:
- **Strategy** – concrete implementations decide how indexing/search is performed.
- **Adapter** – the service abstracts the underlying search engine.
- **Facade** – provides a simple API to the rest of the business layer.

## 2. Detailed Description
The search flow follows a simple lifecycle:

1. **Indexing**  
   - When a product is created or updated, the calling code should invoke `index(store, product)`.  
   - The implementation will convert the product (and optionally its variants) into one or more `Document` objects, then push them to the underlying search index.

2. **Deletion**  
   - When a product is removed, `deleteDocument(store, product)` removes its corresponding document(s) from the index.

3. **Searching**  
   - The client prepares a `SearchRequest` (containing query strings, filters, sorting, etc.) and passes it along with the target store, language, and pagination parameters to `search(...)`.  
   - The service returns a `SearchResponse` containing matched documents and metadata.

4. **Auto‑Complete**  
   - While a user types, `searchKeywords(...)` can be called with the partial input. The service should return a list of suggested keywords that match the prefix.

5. **Direct Document Access**  
   - For debugging or advanced use‑cases, `getDocument(...)` fetches the raw document by its ID.

**Assumptions & Constraints**

- Every product/variant must belong to a `MerchantStore`.  
- `language` is a required parameter; implementations must handle localization (e.g., storing language‑specific fields).  
- The service throws a custom `ServiceException` for all failures, so callers must handle or propagate this exception.  
- Pagination is controlled via `entriesCount` (page size) and `startIndex` (offset).  
- The interface does not define any transaction boundaries; this responsibility lies with the caller or the underlying implementation.

**Architecture Choices**

- The use of a generic `Document` abstraction decouples the service from the specific search engine schema.  
- Optional usage of `Optional<Document>` in `getDocument` signals the possibility of a missing document instead of returning null.  
- The interface is deliberately minimal, focusing only on the operations needed by the application layer; additional features (e.g., bulk indexing, reindexing schedules) would be added in separate interfaces or utility classes.

## 3. Functions/Methods
| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `void index(MerchantStore store, Product product)` | Adds/updates a product (and its variants) in the search index. | **Input:** `store`, `product`. | **Output:** None. | **Effect:** Persists document(s) to the index. |
| `void deleteDocument(MerchantStore store, Product product)` | Removes a product’s document(s) from the index. | **Input:** `store`, `product`. | **Output:** None. | **Effect:** Deletes document(s) from the index. |
| `SearchResponse searchKeywords(MerchantStore store, String language, SearchRequest search, int entriesCount)` | Provides keyword suggestions. | **Input:** `store`, `language`, `search`, `entriesCount`. | **Output:** `SearchResponse` containing suggestion list. | **Effect:** None. |
| `SearchResponse search(MerchantStore store, String language, SearchRequest search, int entriesCount, int startIndex)` | Executes a full product search with pagination. | **Input:** `store`, `language`, `search`, `entriesCount`, `startIndex`. | **Output:** `SearchResponse` with results and metadata. | **Effect:** None. |
| `Optional<Document> getDocument(String language, MerchantStore store, Long id)` | Retrieves a specific indexed document. | **Input:** `language`, `store`, `id`. | **Output:** `Optional<Document>` – empty if not found. | **Effect:** None. |

### Reusable/Utility Methods
- None defined in the interface; concrete classes may expose additional helper methods.

## 4. Dependencies
| Library/Module | Type | Role |
|----------------|------|------|
| `com.salesmanager.core.business.exception.ServiceException` | Third‑party / internal | Custom checked exception for service errors. |
| `com.salesmanager.core.model.catalog.product.Product` | Internal | Domain model for products. |
| `com.salesmanager.core.model.catalog.product.variant.ProductVariant` | Internal | Domain model for product variants. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Internal | Domain model for merchant stores. |
| `modules.commons.search.request.Document` | Third‑party | Generic representation of an indexed document. |
| `modules.commons.search.request.SearchRequest` | Third‑party | Encapsulates search query parameters. |
| `modules.commons.search.request.SearchResponse` | Third‑party | Encapsulates search results and metadata. |

All dependencies are either part of the Sales Manager application or a shared commons module. No external frameworks (e.g., Spring, Hibernate) are referenced directly in this interface.

## 5. Additional Notes
### Edge Cases & Limitations
- **Language Support:** The interface requires a `language` string, but there is no validation or enumeration. Implementations must guard against unsupported or null values.
- **Pagination Bounds:** `startIndex` and `entriesCount` are raw integers; negative values or overly large offsets could cause runtime errors if not validated by the implementation.
- **Missing Product IDs:** `deleteDocument` and `index` accept a `Product` instance but do not expose a method for deleting by ID. Callers must fetch the full product object first.
- **Variant Handling:** While the interface refers to `ProductVariant` in the imports, the `index` method only accepts a `Product`. The responsibility for indexing variants lies within the implementation (e.g., indexing the variants as child documents).

### Potential Enhancements
- **Bulk Operations:** Add `index(List<Product>)`, `deleteDocument(List<Product>)` for batch processing.
- **Reindexing / Refresh:** Expose a `refresh()` method to force index commit or a `reindexAll()` for full rebuild.
- **Metrics & Monitoring:** Provide hooks or callbacks for indexing/search performance metrics.
- **Search Request Builder:** A fluent builder for `SearchRequest` could simplify client code.
- **Error Categorization:** Extend `ServiceException` with specific subclasses (e.g., `IndexingException`, `SearchException`) for finer error handling.

Overall, the interface is concise and well‑documented. Its design promotes flexibility, allowing various search engine backends to be swapped in without impacting the business logic that consumes it.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.search;

import java.util.Optional;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.variant.ProductVariant;
import com.salesmanager.core.model.merchant.MerchantStore;

import modules.commons.search.request.Document;
import modules.commons.search.request.SearchRequest;
import modules.commons.search.request.SearchResponse;

public interface SearchService {
	
	/**
	 * The indexing service for products. The index service must be invoked when a product is
	 * created or updated
	 * @param store
	 * @param product
	 * @throws ServiceException
	 */
	void index(MerchantStore store, Product product) throws ServiceException;

	/**
	 * Deletes a document in the appropriate language. Must be invoked when a product is deleted
	 * @param store
	 * @param product
	 * @throws ServiceException
	 */
	void deleteDocument(MerchantStore store, Product product)
			throws ServiceException;

	/**
	 * Similar keywords based on a a series of characters. Used in the auto-complete
	 */
	SearchResponse searchKeywords(MerchantStore store, String language, SearchRequest search, int entriesCount) throws ServiceException;

	/**
	 * Search products based on user entry
	 */

	SearchResponse search(MerchantStore store, String language, SearchRequest search,
					int entriesCount, int startIndex) throws ServiceException;

	
	Optional<Document> getDocument(String language, MerchantStore store, Long id) throws ServiceException;

}



```
