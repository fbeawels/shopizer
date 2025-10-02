# FileGet.java

## Review

## 1. Summary  

The `FileGet` interface defines a contract for retrieving static CMS content from a file‑based storage backend.  
* **Purpose** – Provide read‑only access to files stored under a merchant store, optionally filtered by a relative path and file type.  
* **Key methods** –  
  * `getFile(...)` – Fetches a single file by name.  
  * `getFileNames(...)` – Returns the names of all files that match the supplied criteria.  
  * `getFiles(...)` – Retrieves the full `OutputContentFile` objects for all matching files.  
* **Design patterns / libraries** – The interface uses Java 8’s `Optional` for the path argument and follows a simple repository‑style pattern (no state, pure read operations). No heavy frameworks are involved; it relies on standard JDK types and the domain model classes `FileContentType` and `OutputContentFile`.

---

## 2. Detailed Description  

### Core Components  
| Component | Role |
|-----------|------|
| `FileGet` | Service contract for file retrieval. |
| `FileContentType` | Enum/DTO that identifies the type of content (e.g. HTML, image, etc.). |
| `OutputContentFile` | DTO that contains the file contents and metadata returned to callers. |

### Execution Flow  
1. **Initialization** – Concrete implementations (e.g. `FileGetImpl`) are typically injected via a dependency‑injection framework (Spring, Guice, etc.).  
2. **Runtime** – A client calls one of the three methods, passing the merchant store code, an optional path, a content type, and (for `getFile`) the file name.  
3. **Implementation** – The implementation resolves the physical location (likely a directory under a store‑specific root), filters by type and optional path, then constructs `OutputContentFile` objects or simply returns file names.  
4. **Exception handling** – All methods declare `ServiceException`, a custom checked exception that indicates problems such as “file not found” or “access denied”.  

### Assumptions & Constraints  
* The path argument is optional; when absent, the root of the store’s file space is assumed.  
* The interface is read‑only; there are no mutation methods.  
* It is expected that the caller has the necessary permissions (handled internally by the implementation).  
* The underlying storage is assumed to be file‑system based; the interface does not expose any persistence details.

### Architecture & Design Choices  
* **Separation of concerns** – Retrieval logic is abstracted away from the rest of the system.  
* **Optional for path** – Makes the API flexible for callers who do or do not want to specify a sub‑folder.  
* **Method granularity** – Provides both name‑only and full‑content retrieval, allowing callers to pick the most efficient operation for their use case.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Parameters | Returns | Exceptions | Side Effects |
|--------|-----------|---------|------------|---------|------------|--------------|
| `getFile` | `OutputContentFile getFile(String merchantStoreCode, Optional<String> path, FileContentType fileContentType, String contentName)` | Retrieve a single file by name. | * `merchantStoreCode` – identifies the store.<br>* `path` – optional relative folder.<br>* `fileContentType` – type filter.<br>* `contentName` – file name. | `OutputContentFile` – full content and metadata. | `ServiceException` – e.g. file missing, access error. | None (read‑only). |
| `getFileNames` | `List<String> getFileNames(String merchantStoreCode, Optional<String> path, FileContentType fileContentType)` | List the names of all matching files. | Same as `getFile` minus `contentName`. | `List<String>` – file names only. | `ServiceException` | None. |
| `getFiles` | `List<OutputContentFile> getFiles(String merchantStoreCode, Optional<String> path, FileContentType fileContentType)` | Retrieve full `OutputContentFile` objects for all matching files. | Same as `getFile` minus `contentName`. | `List<OutputContentFile>` – full content for each file. | `ServiceException` | None. |

### Reusable / Utility Methods  
The interface itself contains no concrete logic, but a typical implementation might offer helper methods such as:

* `buildFilePath(merchantStoreCode, Optional<String> path, String fileName)` – normalizes path separators.  
* `filterByContentType(File file, FileContentType type)` – encapsulates type‑specific filtering.

---

## 4. Dependencies  

| Dependency | Nature | Notes |
|------------|--------|-------|
| Java SE (≥ 8) | Standard | Uses `Optional`, `List`, `String`. |
| `com.salesmanager.core.model.content.FileContentType` | Domain | Enum/DTO defining content categories. |
| `com.salesmanager.core.model.content.OutputContentFile` | Domain | DTO holding file data and metadata. |
| `com.salesmanager.core.business.exception.ServiceException` | Domain | Custom checked exception for service layer errors. |

No external libraries or frameworks are directly referenced; however, implementations are likely to use:

* **Spring Framework** – for dependency injection and transaction management.  
* **Apache Commons IO** – for file manipulation utilities (not shown here).  
* **JUnit / Mockito** – for unit testing.

---

## 5. Additional Notes  

### Strengths  
* **Clear contract** – The interface cleanly separates read operations from any write logic.  
* **Flexibility** – Optional path allows callers to specify sub‑folders or default to the store root.  
* **Explicit exception handling** – Forces callers to handle `ServiceException`, which promotes robust error handling.

### Potential Issues / Edge Cases  
1. **Path Traversal** – Implementations must sanitize the optional path to prevent directory traversal attacks (`..`, absolute paths).  
2. **Case Sensitivity** – File systems differ in case handling; callers should be aware that `contentName` may be case‑sensitive on some platforms.  
3. **Large File Retrieval** – `getFiles` could return many large `OutputContentFile` objects, potentially exhausting memory. A paginated or streaming approach might be safer.  
4. **Null Parameters** – While `Optional` covers `path`, the other parameters (`merchantStoreCode`, `fileContentType`, `contentName`) are not wrapped, so implementations must guard against `null`.  

### Suggested Enhancements  
* **Pagination/Streaming** – Add overloaded methods that accept page size and offset, or return a `Stream<OutputContentFile>` to handle large datasets.  
* **Path Validation API** – Provide a helper method or a separate validator to ensure safe path usage.  
* **Documentation Enhancements** – Expand Javadoc to describe expected path format, case sensitivity, and possible exceptions in more detail.  
* **Return Optional for getFile** – Instead of throwing `ServiceException` for “not found”, return `Optional<OutputContentFile>` to allow callers to handle missing files without exception handling.  
* **Immutable DTOs** – Ensure `OutputContentFile` is immutable or provides defensive copies to avoid accidental modification of the underlying data.  

### Example Extension (Pagination)  
```java
Page<OutputContentFile> getFiles(
        String merchantStoreCode,
        Optional<String> path,
        FileContentType fileContentType,
        int pageNumber,
        int pageSize) throws ServiceException;
```

Overall, the interface is concise and well‑named, making it straightforward for developers to implement and consume. Minor improvements around path safety and handling large data sets would elevate its robustness for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.content;

import java.util.List;
import java.util.Optional;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.OutputContentFile;


/**
 * Methods to retrieve the static content from the CMS
 * 
 * @author Carl Samson
 *
 */
public interface FileGet {

  OutputContentFile getFile(final String merchantStoreCode, Optional<String> path, FileContentType fileContentType,
      String contentName) throws ServiceException;

  List<String> getFileNames(final String merchantStoreCode, Optional<String> path, FileContentType fileContentType)
      throws ServiceException;

  List<OutputContentFile> getFiles(final String merchantStoreCode,
		  Optional<String> path, FileContentType fileContentType) throws ServiceException;
}



```
