# MarketPlaceFacadeImpl.java

## Review

## 1. Summary

`MarketPlaceFacadeImpl` is a Spring‑managed component that bridges between the persistence layer (via `OptinService`) and the API model (`Readable*` DTOs).  
It implements the `MarketPlaceFacade` interface and exposes two main operations:

1. **`get(String store, Language lang)`** – retrieves a `ReadableMerchantStore` for a given store code and language, then wraps it in a `ReadableMarketPlace`.
2. **`findByMerchantAndType(MerchantStore store, OptinType type)`** – fetches an `Optin` entity for the supplied merchant and type, and converts it into a `ReadableOptin` DTO.

The class uses Spring’s `@Component` annotation and dependency injection (`@Inject`) for its collaborators (`StoreFacade` and `OptinService`). It also employs custom runtime exceptions (`ConversionRuntimeException`, `ServiceRuntimeException`) to wrap checked exceptions thrown by underlying services.

## 2. Detailed Description

### Component Overview
```text
MarketPlaceFacadeImpl
├─ dependencies
│   ├─ StoreFacade          (service to fetch store data)
│   └─ OptinService         (service to fetch opt‑in data)
└─ exposed methods
    ├─ get(store, lang)
    └─ findByMerchantAndType(store, type)
```

#### Execution Flow

| Step | Method | Purpose | Key Operations |
|------|--------|---------|----------------|
| 1 | `get(String store, Language lang)` | Entry point for retrieving a readable marketplace | Calls `storeFacade.getByCode(...)` → builds `ReadableMarketPlace` |
| 2 | `createReadableMarketPlace(ReadableMerchantStore)` | Builds DTO | Instantiates `ReadableMarketPlace`, assigns the store |
| 3 | `findByMerchantAndType(MerchantStore, OptinType)` | Entry point for opt‑in lookup | Calls `getOptinByMerchantAndType(...)` → `convertOptinToReadableOptin(...)` |
| 4 | `getOptinByMerchantAndType(...)` | Wrapper around `OptinService` | Calls `optinService.getOptinByMerchantAndType(...)`; throws `ResourceNotFoundException` if null |
| 5 | `convertOptinToReadableOptin(...)` | DTO conversion | Instantiates `ReadableOptinPopulator`, calls `populate(...)`; converts checked exception |

### Design Choices & Assumptions

- **Separation of concerns**: The facade focuses on orchestration while delegating persistence to `OptinService` and data conversion to `ReadableOptinPopulator`.
- **Exception handling**: Checked exceptions from the service layer (`ServiceException`, `ConversionException`) are wrapped in runtime counterparts to avoid propagating checked exceptions through the API layer.
- **Optional usage**: The `Optional` wrapper in `getOptinByMerchantAndType` is a lightweight way to check for null and throw a custom exception.
- **DTO population**: The `ReadableOptinPopulator` is instantiated directly rather than injected, implying it has no state and is inexpensive to create.

### Constraints & Dependencies

- Expects that `StoreFacade.getByCode` and `OptinService.getOptinByMerchantAndType` behave correctly (throwing `ServiceException` on failure).
- Relies on the existence of `ReadableMarketPlace`, `ReadableMerchantStore`, and `ReadableOptin` DTOs, as well as the `ReadableOptinPopulator`.
- Assumes that `ReadableOptinPopulator.populate` accepts `null` for fields that are not relevant in this context.

## 3. Functions/Methods

| Method | Parameters | Return Type | Purpose | Side Effects / Notes |
|--------|------------|-------------|---------|----------------------|
| `get(String store, Language lang)` | `store` (String), `lang` (Language) | `ReadableMarketPlace` | Retrieve a readable marketplace for the given store code and language. | Delegates to `storeFacade`; may throw runtime exceptions if underlying service fails. |
| `createReadableMarketPlace(ReadableMerchantStore readableStore)` | `readableStore` (ReadableMerchantStore) | `ReadableMarketPlace` | Builds a `ReadableMarketPlace` DTO and sets its store. | Currently a stub – TODO indicates future enrichment. |
| `findByMerchantAndType(MerchantStore store, OptinType type)` | `store` (MerchantStore), `type` (OptinType) | `ReadableOptin` | Look up an opt‑in by merchant and type, returning a readable representation. | Throws `ServiceRuntimeException` or `ConversionRuntimeException` if underlying calls fail. |
| `getOptinByMerchantAndType(MerchantStore store, OptinType type)` | `store`, `type` | `Optin` | Helper that fetches the opt‑in entity and wraps missing‑entity logic. | Throws `ResourceNotFoundException` if no opt‑in exists. |
| `convertOptinToReadableOptin(MerchantStore store, Optin optin)` | `store`, `optin` | `ReadableOptin` | Helper that transforms an `Optin` entity into a DTO using `ReadableOptinPopulator`. | Wraps `ConversionException` into a runtime exception. |

### Reusable/Utility Methods
- The two private helpers (`getOptinByMerchantAndType`, `convertOptinToReadableOptin`) encapsulate common patterns (service call + exception wrapping) and could be reused by other methods or classes if needed.

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `StoreFacade` | Spring Bean (`@Inject`) | Provides merchant store lookup. |
| `OptinService` | Spring Bean (`@Inject`) | Provides opt‑in retrieval. |
| `ReadableOptinPopulator` | Plain Java class | Converts `Optin` → `ReadableOptin`; not injected. |
| `ReadableMarketPlace`, `ReadableMerchantStore`, `ReadableOptin` | DTOs | Model objects exposed by the facade. |
| `ConversionRuntimeException`, `ServiceRuntimeException`, `ResourceNotFoundException` | Custom runtime exceptions | Used for wrapping checked exceptions. |
| `Optional` | Java 8+ | Lightweight null‑check wrapper. |
| `@Component`, `@Inject` | Spring Framework | Enables component scanning and dependency injection. |

All dependencies are either Spring‑provided or project‑specific; there are no external third‑party libraries beyond those already part of the Spring ecosystem.

## 5. Additional Notes & Recommendations

### Strengths
- Clear separation of responsibilities.
- Consistent exception handling strategy.
- Easy to test due to dependency injection.

### Potential Improvements
1. **Inject `ReadableOptinPopulator`**  
   Instead of creating a new instance each call, inject a singleton. This also facilitates unit testing (mocking).

2. **Validate Method Arguments**  
   Currently, there is no guard against `null` parameters (e.g., `store` or `lang`). Adding `Objects.requireNonNull` or similar checks would make failures clearer.

3. **Populate `ReadableMarketPlace` Properly**  
   The `TODO` comment indicates missing logic. Ensure that all required fields are populated before returning to the caller.

4. **Logging**  
   Adding SLF4J logging in exception blocks would aid troubleshooting (e.g., log the store code or type that caused the failure).

5. **Error Message Clarity**  
   The `ResourceNotFoundException` message “Option not found” is ambiguous. Consider including the store code and type in the message.

6. **Parameter Naming Consistency**  
   The `findByMerchantAndType` method accepts a `MerchantStore` but internally calls `getOptinByMerchantAndType` which expects a `MerchantStore`. It might be more intuitive to use `store` consistently.

7. **Avoid Null Passes to Populator**  
   Passing `null` for some arguments of `populate` is a code smell. Either overload the populator or design it to accept an options object.

8. **Transactional Boundaries**  
   If the underlying services require transactions, ensure that the facade’s methods are annotated appropriately (e.g., `@Transactional(readOnly = true)`).

9. **Unit Test Coverage**  
   Since this class is thin, unit tests should focus on ensuring correct exception wrapping and that the DTOs are populated as expected.

10. **Documentation**  
    Add Javadoc to public methods to clarify contract, especially for the exception scenarios.

### Edge Cases Not Handled
- **Empty or Whitespace Store Code**: `storeFacade.getByCode` might throw a generic exception. Consider validating the input first.
- **Unsupported Language**: If the language is not supported, the `storeFacade` might return a default or fail; this should be documented.
- **Concurrent Modifications**: If opt‑ins can change concurrently, consider cache‑invalidation strategies or version checks.

### Future Enhancements
- **Caching**: Frequently accessed marketplace data could be cached to reduce load on the `StoreFacade` and `OptinService`.
- **Bulk Retrieval**: Add methods to fetch multiple opt‑ins or marketplaces in one call.
- **Internationalization**: Ensure that DTO fields that are language‑dependent are correctly localized.

Overall, the implementation is clean and follows standard Spring practices. With the recommended refinements—particularly around argument validation, dependency injection, and DTO population—the facade will be more robust, maintainable, and testable.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.marketplace.facade;

import com.salesmanager.core.business.exception.ConversionException;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.shop.store.api.exception.ConversionRuntimeException;
import com.salesmanager.shop.store.api.exception.ResourceNotFoundException;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;
import java.util.Optional;
import javax.inject.Inject;

import org.springframework.stereotype.Component;

import com.salesmanager.core.business.services.system.optin.OptinService;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.system.optin.Optin;
import com.salesmanager.core.model.system.optin.OptinType;
import com.salesmanager.shop.model.marketplace.ReadableMarketPlace;
import com.salesmanager.shop.model.store.ReadableMerchantStore;
import com.salesmanager.shop.model.system.ReadableOptin;
import com.salesmanager.shop.populator.system.ReadableOptinPopulator;
import com.salesmanager.shop.store.controller.store.facade.StoreFacade;

@Component
public class MarketPlaceFacadeImpl implements MarketPlaceFacade {

	@Inject
	private StoreFacade storeFacade;

	@Inject
	private OptinService optinService;

	@Override
	public ReadableMarketPlace get(String store, Language lang) {
		ReadableMerchantStore readableStore = storeFacade.getByCode(store, lang);
    return createReadableMarketPlace(readableStore);
	}

  private ReadableMarketPlace createReadableMarketPlace(ReadableMerchantStore readableStore) {
    //TODO add info from Entity
    ReadableMarketPlace marketPlace = new ReadableMarketPlace();
    marketPlace.setStore(readableStore);
    return marketPlace;
  }

  @Override
	public ReadableOptin findByMerchantAndType(MerchantStore store, OptinType type) {
		Optin optin = getOptinByMerchantAndType(store, type);
    return convertOptinToReadableOptin(store, optin);
	}

  private Optin getOptinByMerchantAndType(MerchantStore store, OptinType type) {
	  try{
      return Optional.ofNullable(optinService.getOptinByMerchantAndType(store, type))
          .orElseThrow(() -> new ResourceNotFoundException("Option not found"));
    } catch (ServiceException e) {
	    throw new ServiceRuntimeException(e);
    }

  }

  private ReadableOptin convertOptinToReadableOptin(MerchantStore store, Optin optin) {
	  try{
      ReadableOptinPopulator populator = new ReadableOptinPopulator();
      return populator.populate(optin, null, store, null);
    } catch (ConversionException e) {
	    throw new ConversionRuntimeException(e);
    }

  }

}



```
