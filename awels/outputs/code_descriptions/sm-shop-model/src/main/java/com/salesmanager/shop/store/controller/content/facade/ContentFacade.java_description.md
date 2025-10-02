# ContentFacade.java

## Review

## 1. Summary
The `ContentFacade` interface defines a contract for managing CMS‑related assets (files, pages, boxes, and folders) in a multi‑tenant e‑commerce application.  
It abstracts the underlying data‑access and business logic for:

* File storage (add, delete, rename, download, path resolution)  
* Page management (CRUD, listing, lookup by code/name)  
* Box management (CRUD, listing, prefix search)  
* Folder retrieval  

Key design points:

* **Facade Pattern** – exposes a simplified API to the presentation layer while hiding the complexity of the underlying content service stack.  
* **DTO‑centric** – all public methods use shop‑level DTOs (`PersistableContentPage`, `ReadableContentBox`, `ContentFile`, etc.) instead of core entity objects, encouraging separation of concerns.  
* **Multi‑store & multi‑language** – every method accepts a `MerchantStore` and `Language`, allowing data isolation per store/language.  
* **Deprecation notice** – the interface contains a deprecated method (`getContent`) indicating a shift from core to shop models.

## 2. Detailed Description
The interface represents a service that will likely be implemented by a Spring component.  
Typical flow:

1. **Initialization** – the implementing class will be instantiated by Spring, with dependencies such as `ContentService`, `FileStorageService`, `MerchantService`, etc.  
2. **Runtime** – the facade receives requests from controllers or other services, delegates to the underlying components, maps core entities to DTOs, and returns the result.  
3. **Cleanup** – no explicit cleanup methods are required; any resources (e.g., file streams) are closed by the implementing code.

### Core Components

| Component | Responsibility |
|-----------|----------------|
| **ContentFolder** | Represents a folder structure in the CMS. |
| **ContentFile** | DTO for file metadata used when adding files. |
| **PersistableContentPage / ReadableContentPage** | Write/read models for content pages. |
| **PersistableContentBox / ReadableContentBox** | Write/read models for content boxes. |
| **OutputContentFile** | Wrapper for file download streams (content type, metadata, stream). |

### Execution Flow Examples

| Operation | Typical Flow |
|-----------|--------------|
| `addContentFile` | Map `ContentFile` to core `FileContent`, persist via `FileContentService`, return. |
| `getContentPages` | Query `PageService` with pagination, convert each entity to `ReadableContentPage`, wrap in `ReadableEntityList`. |
| `download` | Resolve file path via `absolutePath`, open an `InputStream`, populate `OutputContentFile`. |

Assumptions & Constraints:

* Every method can throw a generic `Exception`. In production code, this should be more specific (e.g., `IOException`, `ContentNotFoundException`).  
* The interface expects the caller to provide a valid `MerchantStore` and `Language`; no validation is performed.  
* The file operations assume a file system or cloud storage backend; the implementation must handle concurrency and access control.

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getContentFolder(String folder, MerchantStore store)` | Retrieve a folder tree. | `folder` path, `store` | `ContentFolder` | None |
| `absolutePath(MerchantStore store, String file)` | Resolve absolute path of a file. | `store`, `file` name | `String` path | None |
| `delete(MerchantStore store, String fileName, String fileType)` | Delete a file of a given type. | `store`, `fileName`, `fileType` | void | Removes file from storage |
| `delete(MerchantStore store, Long id)` | Delete a content page by ID. | `store`, `id` | void | Removes page record |
| `getContentPages(...)` | List pages for a store/language with pagination. | `store`, `language`, `page`, `count` | `ReadableEntityList<ReadableContentPage>` | None |
| `getContentPage(String code, MerchantStore store, Language language)` | Fetch a page by code. | `code`, `store`, `language` | `ReadableContentPage` | None |
| `getContentPageByName(String name, MerchantStore store, Language language)` | Fetch a page by name. | `name`, `store`, `language` | `ReadableContentPage` | None |
| `getContentBox(String code, MerchantStore store, Language language)` | Retrieve a box by code. | `code`, `store`, `language` | `ReadableContentBox` | None |
| `codeExist(String code, String type, MerchantStore store)` | Check existence of a content code. | `code`, `type`, `store` | `boolean` | None |
| `getContentBoxes(ContentType type, String codePrefix, MerchantStore store, Language language, int start, int count)` | List boxes filtered by type & prefix. | `type`, `codePrefix`, `store`, `language`, `start`, `count` | `ReadableEntityList<ReadableContentBox>` | None |
| `getContentBoxes(ContentType type, MerchantStore store, Language language, int start, int count)` | List boxes filtered by type. | `type`, `store`, `language`, `start`, `count` | `ReadableEntityList<ReadableContentBox>` | None |
| `addContentFile(ContentFile file, String merchantStoreCode)` | Persist a single file. | `file`, `merchantStoreCode` | void | Stores file metadata & content |
| `addContentFiles(List<ContentFile> file, String merchantStoreCode)` | Persist multiple files. | `file`, `merchantStoreCode` | void | Stores each file |
| `saveContentPage(PersistableContentPage page, MerchantStore merchantStore, Language language)` | Create a new page. | `page`, `store`, `language` | `Long` id | Inserts page record |
| `updateContentPage(Long id, PersistableContentPage page, MerchantStore merchantStore, Language language)` | Update existing page. | `id`, `page`, `store`, `language` | void | Updates record |
| `deleteContent(Long id, MerchantStore merchantStore)` | Delete content by ID. | `id`, `store` | void | Removes record |
| `saveContentBox(PersistableContentBox box, MerchantStore merchantStore, Language language)` | Create a new box. | `box`, `store`, `language` | `Long` id | Inserts box record |
| `updateContentBox(Long id, PersistableContentBox box, MerchantStore merchantStore, Language language)` | Update existing box. | `id`, `box`, `store`, `language` | void | Updates record |
| `getContent(String code, MerchantStore store, Language language)` (Deprecated) | Retrieve full content (old API). | `code`, `store`, `language` | `ReadableContentFull` | None |
| `getContents(Optional<String> type, MerchantStore store, Language language)` | List all contents, optionally filtered by type. | `type`, `store`, `language` | `List<ReadableContentEntity>` | None |
| `renameFile(MerchantStore store, FileContentType fileType, String originalName, String newName)` | Rename a file. | `store`, `fileType`, `originalName`, `newName` | void | Renames in storage |
| `download(MerchantStore store, FileContentType fileType, String fileName)` | Download a file. | `store`, `fileType`, `fileName` | `OutputContentFile` | Provides stream & metadata |

### Reusable / Utility Methods
* `absolutePath` and `renameFile` are low‑level utilities used by higher‑level file operations.  
* Pagination helpers (`start`, `count`) used across listing methods.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `MerchantStore`, `Language` | Core model | Domain objects representing a tenant and language. |
| `ContentType`, `FileContentType`, `OutputContentFile` | Core | Enumerations / wrappers for content metadata. |
| Shop‑layer DTOs (`ContentFile`, `ContentFolder`, `PersistableContentPage`, `ReadableContentPage`, `PersistableContentBox`, `ReadableContentBox`, `ReadableContentEntity`, `ReadableContentFull`) | Shop model | Defined in `com.salesmanager.shop.model`. |
| `ReadableEntityList` | Utility | Generic wrapper providing paging metadata. |
| No explicit third‑party libraries are declared; however, implementation is expected to use Spring (e.g., `@Service`, `@Repository`) and a file storage abstraction (JDK NIO, Amazon S3 SDK, etc.). |

Platform assumptions:

* Java 8+ (stream API, `Optional` usage).  
* Multi‑threaded environment (concurrent requests for file operations).  
* External file system or cloud storage available.

## 5. Additional Notes
### Edge Cases & Missing Features
* **Error handling** – generic `Exception` is too broad; callers cannot differentiate between IO errors, validation failures, or business rule violations.  
* **Transactionality** – operations like `addContentFiles` should be atomic; the interface does not convey this.  
* **File naming conflicts** – `renameFile` does not indicate behavior when `newName` already exists.  
* **Security** – No ACL or permission checks are defined; responsibility must be handled in the implementation.  
* **Locale‑specific content** – Methods return a single `ReadableContentPage` but do not expose a strategy for content overriding per locale.  

### Potential Enhancements
1. **Refactor exception hierarchy** – create domain‑specific exceptions (`ContentNotFoundException`, `FileStorageException`).  
2. **Introduce DTO validation** – use Bean Validation (`@Valid`) on persistable DTOs.  
3. **Add bulk operations with transactional guarantees** – `addContentFiles` could return a list of IDs or a failure report.  
4. **Extend file operations** – provide stream/byte[] getters for in‑memory manipulation; allow chunked uploads.  
5. **Add search capabilities** – e.g., full‑text search on page titles or box content.  
6. **Decouple file path logic** – `absolutePath` could be moved to a dedicated `FilePathResolver` service.  
7. **Deprecate `getContent` gradually** – provide a migration guide and add a replacement method signature.

### Code Style & Maintainability
* The interface contains a TODO comment indicating a shift from core to shop models; the comment should be resolved or removed.  
* Method naming is generally clear, but a few could be more consistent (`delete` overloads).  
* Adding Javadoc to each method (beyond the minimal comment) would improve developer experience.  

Overall, `ContentFacade` provides a clean abstraction for content management in a multi‑tenant CMS. Implementers should focus on robust error handling, transaction safety, and security controls to complement this contract.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.content.facade;

import java.util.List;
import java.util.Optional;

import com.salesmanager.core.model.content.ContentType;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.OutputContentFile;

//TODO above deprecation, use shop model instead of core model

import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.shop.model.content.ContentFile;
import com.salesmanager.shop.model.content.ContentFolder;
import com.salesmanager.shop.model.content.box.PersistableContentBox;
import com.salesmanager.shop.model.content.box.ReadableContentBox;
import com.salesmanager.shop.model.entity.ReadableEntityList;
import com.salesmanager.shop.model.content.ReadableContentEntity;
import com.salesmanager.shop.model.content.ReadableContentFull;
import com.salesmanager.shop.model.content.page.PersistableContentPage;
import com.salesmanager.shop.model.content.page.ReadableContentPage;

/**
 * Images and files management
 * @author carlsamson
 *
 */
public interface ContentFacade {
	
	
	ContentFolder getContentFolder(String folder, MerchantStore store) throws Exception;
	
	/**
	 * File pth
	 * @param store
	 * @param file
	 * @return
	 */
	String absolutePath(MerchantStore store, String file);
	
	/**
	 * Deletes a file from CMS
	 * @param store
	 * @param fileName
	 */
	void delete(MerchantStore store, String fileName, String fileType);
	
	/**
	 * Delete content page
	 * @param store
	 * @param id
	 */
	void delete(MerchantStore store, Long id);

	
	
	/**
	 * Returns page names and urls configured for a given MerchantStore
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ReadableEntityList<ReadableContentPage> getContentPages(MerchantStore store, Language language, int page, int count);
	
	
	/**
	 * Returns page name by code
	 * @param code
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ReadableContentPage getContentPage(String code, MerchantStore store, Language language);
	
	/**
	 * Returns page by name
	 * @param name
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ReadableContentPage getContentPageByName(String name, MerchantStore store, Language language);

	
	/**
	 * Returns a content box for a given code and merchant store
	 * @param code
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ReadableContentBox getContentBox(String code, MerchantStore store, Language language);
	
	
	/**
	 * @param code
	 * @param type
	 * @param store
	 * @return
	 */
	boolean codeExist(String code, String type, MerchantStore store);
	
	
	/**
	 * Returns content boxes created with code prefix
	 * for example return boxes with code starting with <code>_
	 * @param store
	 * @param language
	 * @return
	 * @throws Exception
	 */
	ReadableEntityList<ReadableContentBox> getContentBoxes(ContentType type, String codePrefix, MerchantStore store, Language language, int start, int count);

	ReadableEntityList<ReadableContentBox> getContentBoxes(ContentType type, MerchantStore store, Language language, int start, int count);

	void addContentFile(ContentFile file, String merchantStoreCode);
	
	/**
	 * Add multiple files
	 * @param file
	 * @param merchantStoreCode
	 */
	void addContentFiles(List<ContentFile> file, String merchantStoreCode);
	
	/**
	 * Creates content page
	 * @param page
	 * @param merchantStore
	 * @param language
	 */
	Long saveContentPage(PersistableContentPage page, MerchantStore merchantStore, Language language);
	
	void updateContentPage(Long id, PersistableContentPage page, MerchantStore merchantStore, Language language);
	
	void deleteContent(Long id, MerchantStore merchantStore);
	
	/**
	 * Creates content box
	 * @param box
	 * @param merchantStore
	 * @param language
	 */
	Long saveContentBox(PersistableContentBox box, MerchantStore merchantStore, Language language);
	
	void updateContentBox(Long id, PersistableContentBox box, MerchantStore merchantStore, Language language);

	
	@Deprecated
	ReadableContentFull getContent(String code, MerchantStore store, Language language);
	
	/**
	 * Get all content types
	 * @param type
	 * @param store
	 * @param language
	 * @return
	 */
	List<ReadableContentEntity> getContents(Optional<String> type, MerchantStore store, Language language);

	/**
	 * Rename file
	 * @param store
	 * @param fileType
	 * @param originalName
	 * @param newName
	 */
	void renameFile(MerchantStore store, FileContentType fileType, String originalName, String newName);
	
	/**
	 * Download file
	 * @param store
	 * @param fileType
	 * @param fileName
	 * @return
	 */
	OutputContentFile download(MerchantStore store, FileContentType fileType, String fileName);

}



```
