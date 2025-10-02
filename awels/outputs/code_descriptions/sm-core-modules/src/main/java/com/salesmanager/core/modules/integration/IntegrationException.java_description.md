# IntegrationException.java

## Review

## 1. Summary
- **Purpose**: `IntegrationException` is a custom runtime exception used within the *SalesManager* integration layer to signal errors that occur during data integration or service interaction.
- **Key Components**:
  - **Error Code** – an integer that classifies the exception (e.g., `ERROR_VALIDATION_SAVE`, `TRANSACTION_EXCEPTION`).
  - **Error Fields** – a list of strings that can hold field‑level validation errors.
  - **Constructors** – overloads for wrapping lower‑level exceptions, passing messages, or supplying only an error code.
- **Design Patterns / Libraries**:  
  - Uses **inheritance** from `ServiceException` (a custom exception in the application) to unify error handling across the service layer.  
  - No external frameworks or libraries are directly referenced; the class relies on standard Java (`java.util.List`).

## 2. Detailed Description
- **Initialization**  
  - When an integration error is detected, the code creates an `IntegrationException` instance, supplying the appropriate constructor:
    - `new IntegrationException(IntegrationException.ERROR_VALIDATION_SAVE, "Validation failed")`
    - `new IntegrationException(e)` to wrap an existing exception.
  - The constructor populates `errorCode` and optionally a message.
- **Runtime Behavior**  
  - The exception propagates up the call stack. Higher‑level service or controller code can catch `IntegrationException` (or its super‑type `ServiceException`) and react accordingly:
    - Log the error.
    - Translate into an HTTP error response (e.g., 400 Bad Request for validation errors).
    - Trigger compensating transactions if `TRANSACTION_EXCEPTION` is raised.
- **Cleanup**  
  - No explicit cleanup is required; the exception instance is garbage‑collected after being handled.

### Assumptions & Constraints
- `ServiceException` is expected to provide constructors that accept `String`, `Exception`, etc.  
- The application is single‑threaded or uses proper synchronization if `IntegrationException` instances are shared (unlikely, as exceptions are typically not shared).  
- The error code values are hardcoded; the system relies on these constants for error handling logic.

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `public int getErrorCode()` | Retrieve the numeric error code. | None | `int` | None |
| `public void setErrorCode(int errorCode)` | Set the numeric error code. | `int errorCode` | void | Mutates instance state |
| `public IntegrationException(Exception e)` | Wraps a lower‑level exception. | `Exception e` | N/A | Calls `super(e)` |
| `public IntegrationException(String message, Exception e)` | Wraps a lower‑level exception with a custom message. | `String message`, `Exception e` | N/A | Calls `super(message, e)` |
| `public IntegrationException(int code, String message)` | Creates an exception with a code and message. | `int code`, `String message` | N/A | Sets `errorCode`, calls `super(message)` |
| `public IntegrationException(int code)` | Creates an exception with a code only. | `int code` | N/A | Sets `errorCode` |
| `public IntegrationException(String message)` | Creates an exception with a message only. | `String message` | N/A | Calls `super(message)` |
| `public void setErrorFields(List<String> errorFields)` | Attach a list of problematic fields. | `List<String> errorFields` | void | Mutates instance state |
| `public List<String> getErrorFields()` | Retrieve the attached field list. | None | `List<String>` | None |

### Reusable/Utility Methods
- `setErrorFields` / `getErrorFields` provide a lightweight way to carry detailed validation errors without resorting to separate exception classes.

## 4. Dependencies
- **Java Standard Library**: `java.util.List`, `java.io.Serializable` (via `ServiceException`), `java.lang.Exception`.
- **Application‑Specific**: `com.salesmanager.core.business.exception.ServiceException` – a custom base exception that likely extends `RuntimeException` or `Exception`.

No third‑party or framework dependencies are required for this class.

## 5. Additional Notes
### Edge Cases & Potential Issues
- **Missing Superclass Constructors**: The class assumes that `ServiceException` has matching constructors. If the superclass changes, this code may fail to compile.
- **Null `errorFields`**: If `getErrorFields()` is called before `setErrorFields`, it returns `null`. Consumers should check for `null` to avoid `NullPointerException`.
- **Immutability**: The `errorFields` list is mutable. If the caller modifies the list after passing it to the exception, the internal state will change. Consider defensively copying or wrapping the list (`Collections.unmodifiableList`).
- **Error Code Consistency**: Only two constants are defined; additional codes might be added later. A more robust approach could use an enum (`IntegrationErrorCode`) for type safety.
- **Internationalization**: The message strings are plain; if the application requires i18n, consider using resource bundles or message keys.

### Future Enhancements
- **Enum for Error Codes**: Replace int constants with an `enum` to provide better type safety and documentation.
- **Error Details Object**: Instead of a simple `List<String>`, create a dedicated DTO (`ErrorDetail`) that can hold field name, value, and error message.
- **JSON Serialization**: If exceptions are exposed via REST, add Jackson annotations to control JSON output.
- **Logging Support**: Embed a logger to automatically log the error when the exception is instantiated, reducing boilerplate in higher layers.

Overall, `IntegrationException` is a straightforward, well‑structured exception class that effectively extends the service‑layer error handling hierarchy. With a few minor improvements (immutability, enum usage), it would be even more robust and maintainable.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.modules.integration;

import java.util.List;

import com.salesmanager.core.business.exception.ServiceException;

public class IntegrationException extends ServiceException {
	

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	public static final int ERROR_VALIDATION_SAVE = 100;
	public static final int TRANSACTION_EXCEPTION = 99;
	
	private List<String> errorFields;
	
	private int errorCode = 0;

	public int getErrorCode() {
		return errorCode;
	}

	public void setErrorCode(int errorCode) {
		this.errorCode = errorCode;
	}

	public IntegrationException(Exception e) {
		super(e);
	}
	
	public IntegrationException(String message, Exception e) {
		super(message,e);
	}
	
	public IntegrationException(int code, String message) {
		
		super(message);
		this.errorCode = code;
	}
	
	public IntegrationException(int code) {
		
		this.errorCode = code;
	}

	public IntegrationException(String message) {
		super(message);
	}

	public void setErrorFields(List<String> errorFields) {
		this.errorFields = errorFields;
	}

	public List<String> getErrorFields() {
		return errorFields;
	}

}



```
