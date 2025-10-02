# Mapper.java

## Review

## 1. Summary  
The `Mapper<S, T>` interface defines a lightweight, reusable contract for converting between two object types (`S` → `T`) while taking into account the context of a `MerchantStore` and a `Language`.  
It is typically used in a “mapping” layer where domain entities are transformed into DTOs (or vice‑versa) before being sent to or received from the presentation layer.  

Key components:  

| Component | Role |
|-----------|------|
| `Mapper<S, T>` | Generic interface that enforces two mapping operations |
| `convert(...)` | Creates a new target object from a source |
| `merge(...)`   | Updates an existing target instance with data from the source |
| `MerchantStore` | Contextual information about the store (e.g., store‑specific defaults, currencies, etc.) |
| `Language` | Locale/language information used for i18n of mapped fields |

The design follows the **Mapper** (or **DTO‑Mapper**) pattern – a simple strategy for separating persistence/Domain objects from view‑specific representations. No external frameworks are used; the interface is intentionally minimal to allow any implementation strategy (manual, reflection, MapStruct, ModelMapper, etc.).

---

## 2. Detailed Description  

### Core responsibilities
1. **Decoupling** – Keeps mapping logic out of business or persistence layers, allowing clean separation of concerns.
2. **Context awareness** – By passing `MerchantStore` and `Language`, the mapper can handle store‑specific or localized data during conversion/merge.
3. **Flexibility** – The generic types `S` and `T` mean a single interface can support many mapper implementations.

### Interaction flow
- **Application layer** obtains a mapper (e.g., via dependency injection).
- When translating a domain object to a DTO, it calls `convert(...)`.
- When updating an existing DTO or entity, it calls `merge(...)`.
- The mapper implementation uses `MerchantStore` and `Language` to decide on values such as localized names, default currencies, or store‑specific overrides.

### Assumptions & constraints
- **Null safety**: The interface itself does not specify how nulls are handled. Implementations must decide whether to allow null sources/destinations or throw `NullPointerException`.
- **Thread safety**: No state is enforced; if an implementation holds mutable state, callers must ensure thread safety.
- **Immutability**: The contract does not forbid either mutable or immutable target objects.
- **Performance**: Conversion is assumed to be relatively cheap; heavy transformations should be optimized within the implementation.

### Architecture & design choices
- **Generic interface**: Eliminates the need for multiple specific mapper interfaces; a single implementation can be reused across services.
- **Context parameters**: Explicitly passing context avoids implicit dependencies and makes unit‑testing straightforward.
- **No default methods**: Keeps the interface lean; any common logic must be placed in a base class or a utility.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `convert` | `T convert(S source, MerchantStore store, Language language)` | Creates a **new** instance of type `T` based on `source` while respecting store and language settings. | `S source` – object to convert.<br>`MerchantStore store` – contextual store information.<br>`Language language` – locale for i18n. | `T` – fully populated target object. | None. | Should be pure (no side‑effects). |
| `merge` | `T merge(S source, T destination, MerchantStore store, Language language)` | Updates an **existing** instance `destination` with values from `source`, again considering store/language context. | `S source` – object providing new data.<br>`T destination` – object to update.<br>`MerchantStore store` – contextual store information.<br>`Language language` – locale. | `T` – the same instance (or a new one) after merge. | May mutate `destination`. | Should preserve fields not present in `source` unless explicitly overwritten. |

### Utility / reusable patterns
While the interface itself is minimal, common implementations often provide helper methods such as:
- `copyProperties(Object src, Object dest)` – using reflection or a library like Apache Commons BeanUtils.
- `applyLocalizedValue(Map<Language, String> translations, Language language)` – to fetch a language‑specific string.
These helpers are typically placed in an abstract base class or a utility class, not in the interface.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `MerchantStore` | Domain model | Likely a JPA entity or POJO representing a merchant store. |
| `Language` | Domain model | Represents a language/localization configuration. |
| Java SE | Standard | No external libraries are referenced directly in the interface. |

If an implementation uses libraries (e.g., MapStruct, ModelMapper, BeanUtils), those would be **implementation‑specific** and are not part of this interface contract.

---

## 5. Additional Notes  

### Edge cases & potential pitfalls
1. **Null inputs** – Implementations should define clear behaviour (return null, throw exception, or default to empty objects).
2. **Partial data** – `source` may lack some fields; the mapper must decide whether to leave `destination` untouched or set defaults.
3. **Recursive structures** – If `S` or `T` contain nested objects, infinite recursion could occur if not handled carefully.
4. **Locale fallback** – When `language` is not available, a sensible fallback (e.g., default language) should be defined.

### Documentation & contracts
Adding Javadoc to each method clarifying expected behaviour (e.g., “creates a new immutable DTO” vs “updates the given instance”) would improve clarity for implementers.

### Extensibility
- **BaseMapper** – An abstract class providing default implementations (e.g., shallow copy) could reduce boilerplate.
- **Chaining** – If multiple mappers are needed (e.g., nested conversions), the interface could expose a method to obtain sub‑mappers or use composition.
- **Caching** – For expensive mapping operations, consider integrating a cache strategy within the implementation.

### Suggested improvements
- **Optional return type** – If a conversion can legitimately fail, consider returning `Optional<T>` instead of `null`.
- **Validation** – Add pre‑condition checks for `source` and `destination` (e.g., `Objects.requireNonNull`) to surface issues early.
- **Unit‑test templates** – Provide a base test class that verifies contract invariants (e.g., `merge(source, destination)` retains non‑null destination when source is null).

Overall, the interface is clean, minimal, and expressive. With proper documentation and careful implementation, it can serve as a robust foundation for mapping logic across a multi‑store, multi‑language application.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.mapper;

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;

public interface Mapper<S, T> {

  T convert(S source, MerchantStore store, Language language);
  T merge(S source, T destination, MerchantStore store, Language language);
  

}



```
