# ContentServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`ContentServiceImpl` is the concrete implementation of `ContentService` in a multi‑tenant e‑commerce platform. It manages CRUD operations for `Content` entities (textual, image, static files, etc.) and provides file‑level operations for merchant stores using an underlying Infinispan‑backed cache (`StaticContentFileManager`).  

**Key Components**  
| Component | Role |
|-----------|------|
| `ContentRepository` | JPA repository for persistent `Content` objects |
| `PageContentRepository` | Paging support for content queries |
| `StaticContentFileManager` | Low‑level file manager that stores and retrieves files in an Infinispan tree cache |
| `ContentServiceImpl` | Orchestrates database access and file storage, validates inputs, and exposes higher‑level business methods |

**Notable Patterns & Libraries**  
- **Spring Framework** (`@Service`, `@Autowired`, `@Inject`, `Assert`)  
- **Spring Data JPA** for repository abstractions  
- **Java 8 Optional** for nullable paths  
- **Apache Commons Lang** (`Validate`) and **SLF4J** for logging  
- **Infinispan** (through `StaticContentFileManager`) for distributed caching of media files  

## 2. Detailed Description  

### Initialization  
* Spring autowires `ContentRepository`, `PageContentRepository`, and `StaticContentFileManager`.  
* The constructor delegates to the generic `SalesManagerEntityServiceImpl` base class for standard CRUD operations.  

### Runtime Behaviour  

1. **Content CRUD**  
   - `saveOrUpdate` decides between `update` and `save` based on the presence of an ID.  
   - `delete`, `getById`, and `listByType` delegate directly to JPA repositories.  
   - `listNameByType` and `getBySeUrl` provide specialized query capabilities.

2. **File Management**  
   - `addContentFile`, `addLogo`, `addOptionImage` transform `InputContentFile` objects into appropriate `FileContentType` and then store them via `contentFileManager`.  
   - `addContentFiles` handles batch uploads.  
   - `removeFile`, `removeFiles`, `removeFolder` perform deletions.  
   - `getContentFile`, `getContentFiles`, `getContentFilesNames` provide retrieval of files and metadata.  
   - `renameFile` reads a file, removes the old entry, and re‑writes it under a new name.  

3. **Folder Operations**  
   - `addFolder`, `listFolders`, `removeFolder` use `contentFileManager` to manage directory‑like structures in the cache.  
   - Directory paths are validated with a Linux‑style regex (`isValidLinuxDirectory`).  

4. **Pagination**  
   - `listByType` overloads support paging with `PageRequest`.  

5. **Utility & Validation**  
   - Uses `Assert`/`Validate` for defensive checks.  
   - `isValidLinuxDirectory` ensures paths conform to a simple `/folder/...` pattern.  

### Cleanup  
File streams (`InputContentFile.getFile()`) are closed in finally blocks to avoid resource leaks.  
The underlying cache and database are managed by Spring; explicit cleanup is not required in this class.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `listByType(ContentType, MerchantStore, Language)` | Retrieve all content of a type for a store/language | `contentType`, `store`, `language` | `List<Content>` | None |
| `delete(Content)` | Delete a content entity | `content` | `void` | Removes from DB |
| `getByLanguage(Long, Language)` | Fetch content by ID and language | `id`, `language` | `Content` | None |
| `listByType(List<ContentType>, MerchantStore, Language)` | List content for multiple types | `contentTypes`, `store`, `language` | `List<Content>` | None |
| `listNameByType(List<ContentType>, MerchantStore, Language)` | List only description names | `contentTypes`, `store`, `language` | `List<ContentDescription>` | None |
| `getByCode(String, MerchantStore)` | Find content by code and store | `code`, `store` | `Content` | None |
| `getById(Long)` | Find content by ID | `id` | `Content` | None |
| `saveOrUpdate(Content)` | Persist or merge | `content` | `void` | DB operation |
| `getByCode(String, MerchantStore, Language)` | Find content by code, store, language | `code`, `store`, `language` | `Content` | None |
| `addContentFile(String, InputContentFile)` | Store a single file in cache | `merchantStoreCode`, `contentFile` | `void` | Adds to cache |
| `addLogo(String, InputContentFile)` | Store a logo image | `merchantStoreCode`, `cmsContentImage` | `void` | Adds to cache |
| `addOptionImage(String, InputContentFile)` | Store a property image | `merchantStoreCode`, `cmsContentImage` | `void` | Adds to cache |
| `addImage(String, InputContentFile)` | Internal helper for images | `merchantStoreCode`, `contentImage` | `void` | Adds to cache |
| `addFile(String, InputContentFile)` | Internal helper for files | `merchantStoreCode`, `contentImage` | `void` | Adds to cache |
| `addContentFiles(String, List<InputContentFile>)` | Batch upload | `merchantStoreCode`, `contentFilesList` | `void` | Adds to cache |
| `removeFile(String, FileContentType, String)` | Delete single file | `merchantStoreCode`, `fileType`, `fileName` | `void` | Removes from cache |
| `removeFile(String, String)` | Delete by name (auto‑detect type) | `storeCode`, `fileName` | `void` | Removes from cache |
| `removeFiles(String)` | Delete all files for store | `merchantStoreCode` | `void` | Clears store cache |
| `getContentFile(String, FileContentType, String)` | Retrieve a file | `merchantStoreCode`, `fileType`, `fileName` | `OutputContentFile` | None |
| `getContentFiles(String, FileContentType)` | List files of a type | `merchantStoreCode`, `fileType` | `List<OutputContentFile>` | None |
| `getContentFilesNames(String, FileContentType)` | List file names | `merchantStoreCode`, `fileType` | `List<String>` | None |
| `getBySeUrl(MerchantStore, String)` | Find content by SEO URL | `store`, `seUrl` | `Content` | None |
| `getByCodeLike(ContentType, String, MerchantStore, Language)` | Wild‑card search | `type`, `codeLike`, `store`, `language` | `List<Content>` | None |
| `addFolder(MerchantStore, Optional<String>, String)` | Create folder in cache | `store`, `path`, `folderName` | `void` | Adds folder |
| `listFolders(MerchantStore, Optional<String>)` | List folders | `store`, `path` | `List<String>` | None |
| `removeFolder(MerchantStore, Optional<String>, String)` | Delete folder | `store`, `path`, `folderName` | `void` | Removes folder |
| `isValidLinuxDirectory(String)` | Validate path format | `path` | `boolean` | None |
| `renameFile(String, FileContentType, Optional<String>, String, String)` | Rename a file in cache | `merchantStoreCode`, `fileType`, `path`, `originalName`, `newName` | `void` | Re‑writes file |
| `listByType(ContentType, MerchantStore, int, int)` | Paginated list | `contentType`, `store`, `page`, `count` | `Page<Content>` | None |
| `listByType(ContentType, MerchantStore, Language, int, int)` | Paginated list with language | same | `Page<Content>` | None |
| `exists(String, ContentType, MerchantStore)` | Check existence by code & type | `code`, `type`, `store` | `boolean` | None |

## 4. Dependencies  

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| **Spring Framework** | Third‑party | Core DI, MVC, Data JPA, Validation |
| **Spring Data JPA** | Third‑party | Repository abstractions, paging |
| **Java 8+** | Standard | Optional, streams |
| **Apache Commons Lang** (`Validate`) | Third‑party | Null/empty checks |
| **SLF4J** | Third‑party | Logging |
| **Infinispan** (via `StaticContentFileManager`) | Third‑party | Distributed cache for media |
| **Java IO & NIO** | Standard | File streams |
| **Java Util Regex** | Standard | Path validation |
| **Java Util List / Optional** | Standard | Collections, Optional |

Platform assumptions: runs within a Spring context, relies on a configured JPA `EntityManager` and Infinispan cache for file storage.

## 5. Additional Notes  

### Strengths  
* **Separation of Concerns** – Database and file storage are decoupled via repositories and a dedicated file manager.  
* **Robust Validation** – Uses both `Assert` and `Validate` to guard against nulls.  
* **Pagination Support** – Makes use of Spring Data `PageRequest`.  
* **Convenient Overloads** – Multiple `listByType` signatures for flexible queries.  

### Potential Issues & Edge Cases  

1. **Hard‑coded Path Logic**  
   * `String p = null; Optional<String> path = Optional.ofNullable(p);` is repeated in many methods.  
   * The logic assumes `null` means “root”; however, `null` is passed even when a path is provided (e.g., `addFolder`). This may cause incorrect file placement.  

2. **File Type Detection**  
   * `removeFile(String, String)` infers type via MIME sniffing (`URLConnection.guessContentTypeFromName`).  
   * If the file name does not contain an extension or contains an uncommon extension, the guess may be wrong, leading to missing deletes.  

3. **Error Handling Consistency**  
   * Some methods catch `Exception` broadly and wrap it in `ServiceException`, others let it propagate.  
   * The `renameFile` method does not handle IO errors during read/write – it could leave a dangling reference if the new file creation fails.  

4. **Resource Leak Risk**  
   * While `addImage`/`addFile` close streams in finally, the `renameFile` method uses the `OutputContentFile`'s `ByteArrayOutputStream` directly without closing the stream after writing back.  
   * If the `StaticContentFileManager` internally opens streams, those may need explicit closure.  

5. **Path Validation Regex**  
   * `isValidLinuxDirectory` allows paths starting with `/` followed by one or more groups of `/[a-zA-Z0-9_-]+`.  
   * It fails for relative paths (`folder/subfolder`) or paths containing periods or spaces.  
   * Consider supporting more flexible patterns or delegating to a library.  

6. **Duplicate `getById` Overloads**  
   * Two overloads accept `MerchantStore` and optionally `Language`.  
   * The logic for permission checking (store mismatch) is duplicated. Extract into a helper method.  

7. **Magic Strings**  
   * File type strings like `"IMAGE"` and `"STATIC_FILE"` are hard‑coded; use the enum values directly.  

8. **Concurrent Modifications**  
   * Methods that modify cache (e.g., `addContentFile`, `removeFile`, `renameFile`) are not synchronized.  
   * If two threads operate on the same file concurrently, race conditions may occur. Consider using lock mechanisms provided by Infinispan.  

### Suggested Enhancements  

* **Refactor Path Handling** – Centralize path resolution logic to avoid duplicated `Optional<String>` construction.  
* **Explicit MIME/ContentType Mapping** – Use `FileContentType` enum for all decisions instead of string comparisons.  
* **Use `try-with-resources`** – Simplify stream closing and improve safety.  
* **Consistent Exception Translation** – Create a dedicated exception handler that maps lower‑level exceptions to `ServiceException`.  
* **Unit Tests** – Add tests for all public methods, especially file operations, to catch path and MIME edge cases.  
* **Performance Metrics** – Instrument file operations to monitor cache hit/miss ratios.  
* **Security Checks** – Ensure the file name and path cannot be used for path traversal attacks.  

Overall, the class is a well‑structured service layer that cleanly separates persistence and file management. Addressing the above edge cases and refactoring suggestions would improve robustness, maintainability, and security.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.content;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.URLConnection;
import java.util.List;
import java.util.Optional;
import java.util.regex.Pattern;

import javax.inject.Inject;

import org.apache.commons.lang3.Validate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.util.Assert;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.StaticContentFileManager;
import com.salesmanager.core.business.repositories.content.ContentRepository;
import com.salesmanager.core.business.repositories.content.PageContentRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.content.Content;
import com.salesmanager.core.model.content.ContentDescription;
import com.salesmanager.core.model.content.ContentType;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;

@Service("contentService")
public class ContentServiceImpl extends SalesManagerEntityServiceImpl<Long, Content> implements ContentService {

	private static final Logger LOG = LoggerFactory.getLogger(ContentServiceImpl.class);

	private final ContentRepository contentRepository;
	
	@Autowired
	private PageContentRepository pageContentRepository;
	

	@Inject
	StaticContentFileManager contentFileManager;

	@Inject
	public ContentServiceImpl(ContentRepository contentRepository) {
		super(contentRepository);

		this.contentRepository = contentRepository;
	}

	@Override
	public List<Content> listByType(ContentType contentType, MerchantStore store, Language language)
			throws ServiceException {

		return contentRepository.findByType(contentType, store.getId(), language.getId());
	}

	@Override
	public void delete(Content content) throws ServiceException {

		Content c = this.getById(content.getId());
		super.delete(c);

	}

	@Override
	public Content getByLanguage(Long id, Language language) throws ServiceException {
		return contentRepository.findByIdAndLanguage(id, language.getId());
	}

	@Override
	public List<Content> listByType(List<ContentType> contentType, MerchantStore store, Language language)
			throws ServiceException {

		/*
		 * List<String> contentTypes = new ArrayList<String>(); for (int i = 0;
		 * i < contentType.size(); i++) {
		 * contentTypes.add(contentType.get(i).name()); }
		 */

		return contentRepository.findByTypes(contentType, store.getId(), language.getId());
	}

	@Override
	public List<ContentDescription> listNameByType(List<ContentType> contentType, MerchantStore store,
			Language language) throws ServiceException {

		return contentRepository.listNameByType(contentType, store, language);
	}

	@Override
	public List<Content> listByType(List<ContentType> contentType, MerchantStore store) throws ServiceException {

		return contentRepository.findByTypes(contentType, store.getId());
	}

	@Override
	public Content getByCode(String code, MerchantStore store) throws ServiceException {

		return contentRepository.findByCode(code, store.getId());

	}

	@Override
	public Content getById(Long id) {
		return contentRepository.findOne(id);
	}

	@Override
	public void saveOrUpdate(final Content content) throws ServiceException {

		// save or update (persist and attach entities
		if (content.getId() != null && content.getId() > 0) {
			super.update(content);
		} else {
			super.save(content);
		}

	}

	@Override
	public Content getByCode(String code, MerchantStore store, Language language) throws ServiceException {
		return contentRepository.findByCode(code, store.getId(), language.getId());
	}

	/**
	 * Method responsible for adding content file for given merchant store in
	 * underlying Infinispan tree cache. It will take {@link InputContentFile}
	 * and will store file for given merchant store according to its type. it
	 * can save an image or any type of file (pdf, css, js ...)
	 * 
	 * @param merchantStoreCode
	 *            Merchant store
	 * @param contentFile
	 *            {@link InputContentFile} being stored
	 * @throws ServiceException
	 *             service exception
	 */
	@Override
	public void addContentFile(String merchantStoreCode, InputContentFile contentFile) throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant store Id can not be null");
		Assert.notNull(contentFile, "InputContentFile image can not be null");
		Assert.notNull(contentFile.getFileName(), "InputContentFile.fileName can not be null");
		Assert.notNull(contentFile.getFileContentType(), "InputContentFile.fileContentType can not be null");

		String mimeType = URLConnection.guessContentTypeFromName(contentFile.getFileName());
		contentFile.setMimeType(mimeType);

		if (contentFile.getFileContentType().name().equals(FileContentType.IMAGE.name())
				|| contentFile.getFileContentType().name().equals(FileContentType.STATIC_FILE.name())) {
			addFile(merchantStoreCode, contentFile);
		} else if(contentFile.getFileContentType().name().equals(FileContentType.API_IMAGE.name())) {
			contentFile.setFileContentType(FileContentType.IMAGE);
			addImage(merchantStoreCode, contentFile);
		} else if(contentFile.getFileContentType().name().equals(FileContentType.API_FILE.name())) {
			contentFile.setFileContentType(FileContentType.STATIC_FILE);
			addFile(merchantStoreCode, contentFile);
		} else {
			addImage(merchantStoreCode, contentFile);
		}

	}

	@Override
	public void addLogo(String merchantStoreCode, InputContentFile cmsContentImage) throws ServiceException {

		Assert.notNull(merchantStoreCode, "Merchant store Id can not be null");
		Assert.notNull(cmsContentImage, "CMSContent image can not be null");

		cmsContentImage.setFileContentType(FileContentType.LOGO);
		addImage(merchantStoreCode, cmsContentImage);

	}

	@Override
	public void addOptionImage(String merchantStoreCode, InputContentFile cmsContentImage) throws ServiceException {

		Assert.notNull(merchantStoreCode, "Merchant store Id can not be null");
		Assert.notNull(cmsContentImage, "CMSContent image can not be null");
		cmsContentImage.setFileContentType(FileContentType.PROPERTY);
		addImage(merchantStoreCode, cmsContentImage);

	}

	private void addImage(String merchantStoreCode, InputContentFile contentImage) throws ServiceException {

		try {
			LOG.info("Adding content image for merchant id {}", merchantStoreCode);

			String p = contentImage.getPath();
			Optional<String> path = Optional.ofNullable(p);
			contentFileManager.addFile(merchantStoreCode, path, contentImage);

		} catch (Exception e) {
			LOG.error("Error while trying to convert input stream to buffered image", e);
			throw new ServiceException(e);

		} finally {

			try {
				if (contentImage.getFile() != null) {
					contentImage.getFile().close();
				}
			} catch (Exception ignore) {
			}

		}

	}

	private void addFile(final String merchantStoreCode, InputContentFile contentImage) throws ServiceException {

		try {
			LOG.info("Adding content file for merchant id {}", merchantStoreCode);
			// staticContentFileManager.addFile(merchantStoreCode,
			// contentImage);

			String p = null;
			Optional<String> path = Optional.ofNullable(p);

			contentFileManager.addFile(merchantStoreCode, path, contentImage);

		} catch (Exception e) {
			LOG.error("Error while trying to convert input stream to buffered image", e);
			throw new ServiceException(e);

		} finally {

			try {
				if (contentImage.getFile() != null) {
					contentImage.getFile().close();
				}
			} catch (Exception ignore) {
			}
		}

	}

	/**
	 * Method responsible for adding list of content images for given merchant
	 * store in underlying Infinispan tree cache. It will take list of
	 * {@link CMSContentImage} and will store them for given merchant store.
	 * 
	 * @param merchantStoreCode
	 *            Merchant store
	 * @param contentImagesList
	 *            list of {@link CMSContentImage} being stored
	 * @throws ServiceException
	 *             service exception
	 */
	@Override
	public void addContentFiles(String merchantStoreCode, List<InputContentFile> contentFilesList)
			throws ServiceException {

		Assert.notNull(merchantStoreCode, "Merchant store ID can not be null");
		Assert.notEmpty(contentFilesList, "File list can not be empty");
		LOG.info("Adding total {} images for given merchant", contentFilesList.size());

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		LOG.info("Adding content images for merchant....");
		contentFileManager.addFiles(merchantStoreCode, path, contentFilesList);
		// staticContentFileManager.addFiles(merchantStoreCode,
		// contentFilesList);

		try {
			for (InputContentFile file : contentFilesList) {
				if (file.getFile() != null) {
					file.getFile().close();
				}
			}
		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	/**
	 * Method to remove given content image.Images are stored in underlying
	 * system based on there name. Name will be used to search given image for
	 * removal
	 * 
	 * @param contentImage
	 * @param merchantStoreCode
	 *            merchant store
	 * @throws ServiceException
	 */
	@Override
	public void removeFile(String merchantStoreCode, FileContentType fileContentType, String fileName)
			throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant Store Id can not be null");
		Assert.notNull(fileContentType, "Content file type can not be null");
		Assert.notNull(fileName, "Content Image type can not be null");

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		contentFileManager.removeFile(merchantStoreCode, fileContentType, fileName, path);

	}

	@Override
	public void removeFile(String storeCode, String fileName) throws ServiceException {

		String fileType = "IMAGE";
		String mimetype = URLConnection.guessContentTypeFromName(fileName);
		String type = mimetype.split("/")[0];
		if (!type.equals("image"))
			fileType = "STATIC_FILE";

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		contentFileManager.removeFile(storeCode, FileContentType.valueOf(fileType), fileName, path);

	}

	/**
	 * Method to remove all images for a given merchant.It will take merchant
	 * store as an input and will remove all images associated with given
	 * merchant store.
	 * 
	 * @param merchantStoreCode
	 * @throws ServiceException
	 */
	@Override
	public void removeFiles(String merchantStoreCode) throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant Store Id can not be null");

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		contentFileManager.removeFiles(merchantStoreCode, path);
	}

	/**
	 * Implementation for getContentImage method defined in
	 * {@link ContentService} interface. Methods will return Content image with
	 * given image name for the Merchant store or will return null if no image
	 * with given name found for requested Merchant Store in Infinispan tree
	 * cache.
	 * 
	 * @param store
	 *            Merchant merchantStoreCode
	 * @param imageName
	 *            name of requested image
	 * @return {@link OutputContentImage}
	 * @throws ServiceException
	 */
	@Override
	public OutputContentFile getContentFile(String merchantStoreCode, FileContentType fileContentType, String fileName)
			throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant store ID can not be null");
		Assert.notNull(fileName, "File name can not be null");

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		return contentFileManager.getFile(merchantStoreCode, path, fileContentType, fileName);

	}

	/**
	 * Implementation for getContentImages method defined in
	 * {@link ContentService} interface. Methods will return list of all Content
	 * image associated with given Merchant store or will return empty list if
	 * no image is associated with given Merchant Store in Infinispan tree
	 * cache.
	 * 
	 * @param merchantStoreId
	 *            Merchant store
	 * @return list of {@link OutputContentImage}
	 * @throws ServiceException
	 */
	@Override
	public List<OutputContentFile> getContentFiles(String merchantStoreCode, FileContentType fileContentType)
			throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant store Id can not be null");
		// return staticContentFileManager.getFiles(merchantStoreCode,
		// fileContentType);
		String p = null;
		Optional<String> path = Optional.ofNullable(p);
		return contentFileManager.getFiles(merchantStoreCode, path, fileContentType);
	}

	/**
	 * Returns the image names for a given merchant and store
	 * 
	 * @param merchantStoreCode
	 * @param imageContentType
	 * @return images name list
	 * @throws ServiceException
	 */
	@Override
	public List<String> getContentFilesNames(String merchantStoreCode, FileContentType fileContentType)
			throws ServiceException {
		Assert.notNull(merchantStoreCode, "Merchant store Id can not be null");

		String p = null;
		Optional<String> path = Optional.ofNullable(p);

		return contentFileManager.getFileNames(merchantStoreCode, path, fileContentType);

		/*
		 * if(fileContentType.name().equals(FileContentType.IMAGE.name()) ||
		 * fileContentType.name().equals(FileContentType.STATIC_FILE.name())) {
		 * return contentFileManager.getFileNames(merchantStoreCode,
		 * fileContentType); } else { return
		 * contentFileManager.getFileNames(merchantStoreCode, fileContentType);
		 * }
		 */
	}

	@Override
	public ContentDescription getBySeUrl(MerchantStore store, String seUrl) {
		return contentRepository.getBySeUrl(store, seUrl);
	}

	@Override
	public List<Content> getByCodeLike(ContentType type, String codeLike, MerchantStore store, Language language) {
		return contentRepository.findByCodeLike(type, '%' + codeLike + '%', store.getId(), language.getId());
	}

	@Override
	public Content getById(Long id, MerchantStore store, Language language) throws ServiceException {

		Content content = contentRepository.findOne(id);

		if (content != null) {
			if (content.getMerchantStore().getId().intValue() != store.getId().intValue()) {
				return null;
			}
		}

		return content;
	}
	
	public Content getById(Long id, MerchantStore store) throws ServiceException {

		Content content = contentRepository.findOne(id);

		if (content != null) {
			if (content.getMerchantStore().getId().intValue() != store.getId().intValue()) {
				return null;
			}
		}

		return content;
	}

	@Override
	public void addFolder(MerchantStore store, Optional<String> path, String folderName) throws ServiceException {
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notNull(folderName, "Folder name cannot be null");
		
		if(path.isPresent()) {
			if(!this.isValidLinuxDirectory(path.get())) {
				throw new ServiceException("Path format [" + path.get() + "] not a valid directory format");
			}
		}
		contentFileManager.addFolder(store.getCode(), folderName, path);


	}

	@Override
	public List<String> listFolders(MerchantStore store, Optional<String> path) throws ServiceException {
		Validate.notNull(store, "MerchantStore cannot be null");
		
		return contentFileManager.listFolders(store.getCode(), path);
	}

	@Override
	public void removeFolder(MerchantStore store, Optional<String> path, String folderName) throws ServiceException {
		Validate.notNull(store, "MerchantStore cannot be null");
		Validate.notNull(folderName, "Folder name cannot be null");
		
		contentFileManager.removeFolder(store.getCode(), folderName, path);

	}
	
	public boolean isValidLinuxDirectory(String path) {
	    Pattern linuxDirectoryPattern = Pattern.compile("^/|(/[a-zA-Z0-9_-]+)+$");
	     return path != null && !path.trim().isEmpty() && linuxDirectoryPattern.matcher( path ).matches();
	}

	@Override
	public void renameFile(String merchantStoreCode, FileContentType fileContentType, Optional<String> path,
			String originalName, String newName) throws ServiceException{

		OutputContentFile file = contentFileManager.getFile(merchantStoreCode, path, fileContentType, originalName);
		
		if(file == null) {
			throw new ServiceException("File name [" + originalName + "] not found for merchant [" + merchantStoreCode +"]");
		}
		
		ByteArrayOutputStream os = file.getFile();
		InputStream is = new ByteArrayInputStream(os.toByteArray());
		
		//remove file
		contentFileManager.removeFile(merchantStoreCode, fileContentType, originalName, path);
		
		//recreate file
		InputContentFile inputFile = new InputContentFile();
		inputFile.setFileContentType(fileContentType);
		inputFile.setFileName(newName);
		inputFile.setMimeType(file.getMimeType());
		inputFile.setFile(is);
		
		contentFileManager.addFile(merchantStoreCode, path, inputFile);
	
	}

	@Override
	public Page<Content> listByType(ContentType contentType, MerchantStore store, int page, int count)
			throws ServiceException {
		Pageable pageRequest = PageRequest.of(page, count);
		return pageContentRepository.findByContentType(contentType, store.getId(), pageRequest);
	}

	@Override
	public Page<Content> listByType(ContentType contentType, MerchantStore store, Language language, int page,
			int count) throws ServiceException {
		Pageable pageRequest = PageRequest.of(page, count);
		return pageContentRepository.findByContentType(contentType, store.getId(), language.getId(), pageRequest);
	}

	@Override
	public boolean exists(String code, ContentType type, MerchantStore store) {
		Content c = contentRepository.findByCodeAndType(code, type, store.getId());
		return c !=null ? true:false;
	}

}



```
