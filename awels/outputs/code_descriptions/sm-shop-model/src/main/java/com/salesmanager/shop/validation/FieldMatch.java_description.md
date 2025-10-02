# FieldMatch.java

## Review

## 1. Summary  

The code defines a custom validation annotation, `@FieldMatch`, for use with the Java Bean Validation API (JSR‑380).  
Its purpose is to enforce that two fields of a bean contain identical values – a typical “password / confirm password” or “email / confirm email” pattern.  

Key components  

| Component | Role |
|-----------|------|
| `@FieldMatch` annotation | Marks a class (or another annotation) with a pair of field names (`first`, `second`) and a message |
| `FieldMatchValidator` | The actual validator class (referenced by `@Constraint`) that implements the comparison logic (implementation not shown) |
| `FieldMatch.List` | A container annotation that allows multiple `@FieldMatch` constraints on the same class |

Design patterns / frameworks  
* **Annotation‑Driven Validation** – Uses the standard `javax.validation` framework.  
* **Container Annotation** – Implements the *Repeatable* pattern via a nested `List` annotation to support multiple pairs.

---

## 2. Detailed Description  

### Flow of Execution  

1. **Annotation application** – A developer applies `@FieldMatch` (or its container) to a DTO or entity class, specifying the two property names that must match.  
2. **Bean Validation bootstrap** – At runtime, a `ValidatorFactory` is created (typically by the application container).  
3. **Constraint discovery** – The validator factory scans the class for constraints. When it encounters `@FieldMatch`, it instantiates the `FieldMatchValidator`.  
4. **Validation** – For each `FieldMatch` instance, `FieldMatchValidator.isValid()` is called with the bean instance.  
5. **Result** – If any pair fails, the validator returns `false` and the message is collected in the `ConstraintViolation` set.  
6. **Cleanup** – Validator instances are typically stateless; no explicit cleanup is required.

### Core assumptions & constraints  

* The bean must expose the specified fields via JavaBeans getter methods (or be accessible through reflection).  
* The validator must handle `null` values – either by treating them as equal or by defining a separate `@NotNull` constraint.  
* The annotation is only applicable to types (`TYPE`) and other annotations (`ANNOTATION_TYPE`), not to fields or methods.

### Architecture & design choices  

* **Decoupled validator** – `FieldMatch` only declares metadata; the actual comparison logic lives in a separate `FieldMatchValidator` class, keeping the annotation lightweight.  
* **Repeatable constraints** – The nested `List` interface follows the standard approach for repeatable annotations in Java 8+, enabling multiple field‑matching rules on the same class without requiring a custom `@Repeatable` meta‑annotation.  
* **Minimal API** – Only the essential parameters (`first`, `second`, `message`, `groups`, `payload`) are exposed, making the annotation straightforward to use.

---

## 3. Functions/Methods  

| Method | Description | Inputs | Outputs | Side Effects |
|--------|-------------|--------|---------|--------------|
| `String message()` | Returns the validation message template. | None | `String` | None |
| `Class<?>[] groups()` | Returns validation groups for this constraint. | None | `Class<?>[]` | None |
| `Class<? extends Payload>[] payload()` | Returns payload metadata for client‑specific information. | None | `Class<? extends Payload>[]` | None |
| `String first()` | Name of the first field to compare. | None | `String` | None |
| `String second()` | Name of the second field to compare. | None | `String` | None |
| `FieldMatch[] value()` (inside `@interface List`) | Holds multiple `FieldMatch` annotations for repeatable usage. | None | `FieldMatch[]` | None |

All methods are **pure** and only return constant values defined in the annotation instance. No mutable state or side effects exist.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.validation.Constraint` | Third‑party (JSR‑380) | Core Bean Validation API |
| `javax.validation.Payload` | Third‑party | Used for attaching metadata to constraints |
| `java.lang.annotation.*` | Standard | Annotation meta‑annotations (`@Documented`, `@Retention`, `@Target`) |
| `FieldMatchValidator` | Project-specific | Custom validator class (implementation not provided) |
| `static java.lang.annotation.ElementType.*` & `static java.lang.annotation.RetentionPolicy.RUNTIME` | Standard | Simplify meta‑annotation usage |

The code is fully portable across any Java environment that supplies the Bean Validation API (e.g., Hibernate Validator, Spring Validation).

---

## 5. Additional Notes  

### Edge‑case handling  
* **Null values** – The validator should decide whether `null == null` is acceptable. If the business rule requires non‑null confirmation fields, an additional `@NotNull` constraint is necessary.  
* **Non‑String fields** – The annotation name (`first`, `second`) implies string fields, but the validator can be generic. Ensure the validator casts or converts the field values appropriately.  
* **Reflection access** – If the fields are private and no getters exist, the validator must access them via reflection or be configured with an `AccessibleObject`.  

### Potential improvements  
1. **Generic type safety** – Use `@Target(ElementType.TYPE)` only; `ANNOTATION_TYPE` is unnecessary unless you plan to create meta‑annotations.  
2. **Custom error codes** – Expose an `errorCode` attribute for integration with i18n systems.  
3. **Automatic payload usage** – Provide a default `Payload` implementation to carry the names of the mismatched fields.  
4. **Documentation examples** – Add Javadoc examples for the container annotation usage (currently commented out).  
5. **Unit tests** – Include tests that cover:
   * Matching values
   * Different values
   * Null handling
   * Multiple `FieldMatch` constraints on one class

### Future extensions  
* **Collection field support** – Extend the validator to compare lists or maps (e.g., same size and element equality).  
* **Custom comparison logic** – Allow a custom `Comparator` provider via the annotation.  
* **Integration with Spring** – Expose a `@FieldMatch` that can be used directly in Spring MVC controller DTOs, automatically wiring the validator through Spring's `LocalValidatorFactoryBean`.

---

## Code Critique



## Code Preview

```java
/**
 * 
 */
package com.salesmanager.shop.validation;

import static java.lang.annotation.ElementType.ANNOTATION_TYPE;
import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import javax.validation.Constraint;
import javax.validation.Payload;


/**
 * 
 * Validation annotation to validate that 2 fields have the same value.
 * An array of fields and their matching confirmation fields can be supplied.
 *
 * Example, compare 1 pair of fields:
 * @FieldMatch(first = "password", second = "confirmPassword", message = "The password fields must match")
 * 
 * Example, compare more than 1 pair of fields:
 * @FieldMatch.List({
 * @FieldMatch(first = "password", second = "confirmPassword", message = "The password fields must match"),
 * @FieldMatch(first = "email", second = "confirmEmail", message = "The email fields must match")})
 * 
 * @author Umesh Awasthi
 * 
 */

@Constraint(validatedBy = FieldMatchValidator.class)
@Documented
@Target({TYPE, ANNOTATION_TYPE})
@Retention(RUNTIME)
public @interface FieldMatch
{

    String message() default "Fields are not matching";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    /**
     * @return The first field
     */
    String first();

    /**
     * @return The second field
     */
    String second();

    /**
     * Defines several <code>@FieldMatch</code> annotations on the same element
     *
     * @see FieldMatch
     */
    @Target({TYPE, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
            @interface List
    {
        FieldMatch[] value();
    }
}



```
