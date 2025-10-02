# AbstractController.java

## Review

## 1. Summary  

The **`AbstractController`** class serves as a lightweight base class for all MVC controllers in the SalesManager shop module.  
It centralises common web‑layer responsibilities such as:

* **Session attribute handling** – generic getters/setters/removals.
* **Locale extraction** – retrieving the current `Language` from the request.
* **Pagination support** – creating and updating `PaginationData` objects.

The class relies on a handful of internal utilities (`SessionUtil`, `Constants`, `PaginationData`) and standard Servlet API classes. No heavyweight frameworks are involved; it is essentially a convenience façade that keeps controller code concise.

---

## 2. Detailed Description  

### Core Components  
| Component | Purpose |
|-----------|---------|
| `getSessionAttribute()` | Retrieves a typed value from the HTTP session. |
| `setSessionAttribute()` | Persists a value into the HTTP session. |
| `removeAttribute()` | Deletes a session attribute. |
| `getLanguage()` | Fetches the `Language` object that has been stored on the request scope (typically by a filter). |
| `createPaginaionData()` | Instantiates a `PaginationData` object given page number/size. |
| `calculatePaginaionData()` | Updates a `PaginationData` instance with total/offset/count information based on the query result set. |

### Execution Flow  

1. **Initialization** – An MVC controller extends `AbstractController`; no explicit constructor logic exists.  
2. **Runtime** – Whenever a controller action needs session data or pagination helpers, it calls the protected methods.  
3. **Cleanup** – No resources are held; the class is stateless so no cleanup is required.  

### Assumptions & Constraints  

* The `SessionUtil` methods are assumed to exist and correctly wrap the raw `HttpSession` API.  
* The request will always carry a `Language` object under the key defined by `Constants.LANGUAGE`.  
* `PaginationData` expects the constructor signature `(pageSize, pageNumber)` – the code uses that order.  
* All methods are **static** in nature (they delegate to static utilities) but are provided as instance methods to enable easy mocking.

### Design Choices  

* **Convenience façade** – By centralising session logic, controllers stay thin.  
* **Generic typing** – `getSessionAttribute` uses a type parameter, but unchecked casting is required because session attributes are stored as raw `Object`.  
* **Method naming** – The word “paginaion” is a misspelling that propagates through the class.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `protected <T> T getSessionAttribute(String key, HttpServletRequest request)` | Generic getter | Retrieves an attribute from the session and casts it to the requested type. | `key` – attribute name, `request` – current HTTP request | Casted attribute value or `null` if missing | None (read‑only) |
| `protected void setSessionAttribute(String key, Object value, HttpServletRequest request)` | Setter | Stores a value in the session under the given key. | `key`, `value`, `request` | None | Writes to session |
| `protected void removeAttribute(String key, HttpServletRequest request)` | Deleter | Removes the session attribute for the given key. | `key`, `request` | None | Deletes from session |
| `protected Language getLanguage(HttpServletRequest request)` | Locale extractor | Obtains the `Language` object stored on the request scope. | `request` | `Language` instance or `null` | None |
| `protected PaginationData createPaginaionData(int pageNumber, int pageSize)` | Pagination factory | Instantiates `PaginationData` with the supplied page size and number. | `pageNumber`, `pageSize` | New `PaginationData` | None |
| `protected PaginationData calculatePaginaionData(PaginationData paginationData, int pageSize, int resultCount)` | Pagination calculator | Updates the supplied `PaginationData` with `totalCount`, `countByPage`, and adjusts `currentPage`. | `paginationData`, `pageSize`, `resultCount` | Updated `PaginationData` | Modifies the passed object |

**Reusable/Utility Methods**  
All methods are `protected`, making them readily usable by subclasses and mockable in unit tests.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | **Standard** | Provides request/session handling. |
| `com.salesmanager.core.model.reference.language.Language` | **Third‑party** | Domain model representing a language. |
| `com.salesmanager.shop.constants.Constants` | **Third‑party** | Holds static configuration such as request attribute keys. |
| `com.salesmanager.shop.store.model.paging.PaginationData` | **Third‑party** | Holds pagination metadata. |
| `com.salesmanager.shop.utils.SessionUtil` | **Third‑party** | Wraps session operations (likely delegating to `HttpSession`). |

All dependencies are internal to the SalesManager project; no external frameworks (Spring, CDI, etc.) are referenced.

---

## 5. Additional Notes  

### Typos & Naming  
* The methods `createPaginaionData` and `calculatePaginaionData` contain a spelling mistake (`paginaion`). Renaming to `createPaginationData` and `calculatePaginationData` would improve readability and avoid confusion when generating documentation.

### Type‑Safety  
* `getSessionAttribute` uses unchecked casting. While convenient, it opens the door to `ClassCastException` if the caller’s expected type does not match the actual stored value.  
  * **Mitigation** – Return an `Optional<T>` or throw a custom exception with a helpful message.

### Pagination Logic  
* `int count = Math.min((currentPage * pageSize), resultCount);`  
  * This logic calculates the **maximum offset** that can be displayed, not the number of items on the current page.  
  * The method also sets `paginationData.setCountByPage(count);`, which may be mis‑named.  
  * A clearer implementation would compute `int itemsOnPage = Math.min(pageSize, resultCount - ((currentPage - 1) * pageSize));`.

### Null & Boundary Checks  
* None of the methods perform null checks on `request` or `key`.  
  * Passing a null request will cause a `NullPointerException`.  
  * A defensive check (or letting the exception surface) should be documented.

### Thread Safety  
* The class is stateless; all operations are local to the request. Therefore, it is inherently thread‑safe in a typical servlet container.

### Testing  
* Because methods delegate to static utilities, unit tests can mock `SessionUtil` via a wrapper or use PowerMock/Mockito to capture static calls.  
* Consider extracting the session operations into an interface (`SessionProvider`) and injecting it; this would make mocking straightforward without relying on static methods.

### Future Enhancements  
1. **Centralised Locale Handling** – Move `getLanguage` to a dedicated `LocaleResolver` component.  
2. **Pagination Utilities** – Provide static helpers that can calculate pagination metadata given total count and page size, so controllers don't need to instantiate or mutate objects.  
3. **Session Abstraction** – Replace direct use of `HttpServletRequest` with a higher‑level session abstraction to improve testability and flexibility (e.g., support for non‑HTTP session stores).  
4. **Documentation** – Add full Javadoc for each method, including parameter and return descriptions, and specify the behaviour when keys are absent.

--- 

**Overall Assessment**  
The class offers a tidy set of helper methods that reduce boilerplate in controllers. Minor improvements—correcting typos, tightening type safety, clarifying pagination semantics—will make the code more robust and maintainable. The reliance on internal utilities is acceptable, but consider refactoring to reduce static dependencies for better testability.

## Code Critique



## Code Preview

```java
/**
 *
 */
package com.salesmanager.shop.store.controller;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.constants.Constants;
import com.salesmanager.shop.store.model.paging.PaginationData;

/**
 * @author Umesh A
 *
 */
public abstract class AbstractController {


    /**
     * Method which will help to retrieving values from Session
     * based on the key being passed to the method.
     * @param key
     * @return value stored in session corresponding to the key
     */
    @SuppressWarnings( "unchecked" )
    protected <T> T getSessionAttribute(final String key, HttpServletRequest request) {
	          return (T) com.salesmanager.shop.utils.SessionUtil.getSessionAttribute(key, request);

	}
    
    protected void setSessionAttribute(final String key, final Object value, HttpServletRequest request) {
    	com.salesmanager.shop.utils.SessionUtil.setSessionAttribute(key, value, request);
	}
    
    
    protected void removeAttribute(final String key, HttpServletRequest request) {
    	com.salesmanager.shop.utils.SessionUtil.removeSessionAttribute(key, request);
	}
    
    protected Language getLanguage(HttpServletRequest request) {
    	return (Language)request.getAttribute(Constants.LANGUAGE);
    }

    protected PaginationData createPaginaionData( final int pageNumber, final int pageSize )
    {
        final PaginationData paginaionData = new PaginationData(pageSize,pageNumber);
       
        return paginaionData;
    }
    
    protected PaginationData calculatePaginaionData( final PaginationData paginationData, final int pageSize, final int resultCount){
        
    	int currentPage = paginationData.getCurrentPage();


    	int count = Math.min((currentPage * pageSize), resultCount);  
    	paginationData.setCountByPage(count);

    	paginationData.setTotalCount( resultCount );
        return paginationData;
    }
}



```
