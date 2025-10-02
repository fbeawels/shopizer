# CmsStaticContentFileManagerImpl.java

## Review

## 1. Summary  

`CmsStaticContentFileManagerImpl` is a concrete implementation of the `ContentAssetsManager` interface that stores and manages static files (images, CSS/JS, PDFs, etc.) on a local file system rather than in an in‑memory cache.  
The class builds a file‑system hierarchy that looks like:

```
<rootName>/<ROOT_CONTAINER>/<merchantStoreCode>/<FileContentType>/<file>
```

Key responsibilities:

| Responsibility | Where it lives | Notes |
|----------------|----------------|-------|
| **Add a single file** | `addFile()` | Copies the supplied `InputStream` to the target path. |
| **Add many files** | `addFiles()` | Loops over a list of `InputContentFile` objects, copying each to disk. |
| **Delete a file** | `removeFile()` | Deletes a single file. |
| **Delete all files for a merchant** | `removeFiles()` | Deletes the merchant directory (though it only calls `deleteIfExists`). |
| **List file names** | `getFileNames()` | Returns the names of files under a particular `FileContentType` directory. |
| **Folder operations** | `addFolder()`, `removeFolder()` | Create / delete sub‑folders under a merchant node. |

The class also keeps a reference to a `LocalCacheManagerImpl` (which it casts to `CMSManager` in `@PostConstruct`) and exposes a static singleton via `getInstance()`.  
A few methods (`getFile`, `getFiles`, `listFolders`, `getCmsManager`) are intentionally unimplemented and simply throw a `ServiceException`.

The implementation relies heavily on `java.nio.file` APIs, `org.apache.commons.lang3.StringUtils`, and SLF4J for logging.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

1. **Initialization**  
   * A static instance is created lazily in `getInstance()`.  
   * The `@PostConstruct` method `init()` is expected to run after dependency injection; it sets `rootName` from the injected `cacheManager` (cast to `CMSManager`).  
   * **Problem**: When using the static factory, `init()` will *not* be called (unless a container explicitly triggers it), leaving `rootName` as the default `"static"`.

2. **Adding Files**  
   * `addFile()` and `addFiles()` both build the root path via `buildRootPath()`.  
   * They create the merchant directory, then the sub‑directory for the `FileContentType`.  
   * The file is copied from the supplied `InputStream` using `Files.copy(..., StandardCopyOption.REPLACE_EXISTING)`.  
   * Logging indicates success; any exception is wrapped in a `ServiceException`.

3. **Removing Files/Folders**  
   * `removeFile()` builds a path to the target file and calls `Files.deleteIfExists()`.  
   * `removeFiles()` deletes the merchant directory path (not recursive).  
   * `removeFolder()` deletes a specific sub‑folder, again non‑recursive.  
   * These methods assume the directories exist; if they are nested more than one level deep, they will silently fail.

4. **Listing Files**  
   * `getFileNames()` enumerates the contents of the `<FileContentType>` directory.  
   * For `IMAGE` types it uses `URLConnection.guessContentTypeFromName` to filter only image files.

5. **Folder Helpers**  
   * `buildMerchantPath()` constructs the merchant base path.  
   * `addFolder()` and `removeFolder()` rely on that helper to locate the correct parent directory.

### 2.2 Design Choices & Assumptions  

| Decision | Rationale | Observed Issues |
|----------|-----------|-----------------|
| **File‑system storage** | Keeps static assets close to the web server for fast access. | Requires proper permission handling, path validation, and cleanup. |
| **Singleton pattern** | Provides a single global instance. | Not thread‑safe; `@PostConstruct` may never run. |
| **Use of `Optional<String> folderPath`** | Intended to allow relative folder specification. | Not used in any method, leading to dead code. |
| **String concatenation via `StringBuilder`** | Simple and explicit. | Repeated manual path construction is error‑prone; better to use `Path.resolve()`. |
| **`Files.createDirectory`** | Creates a single directory. | Fails if parent directories are missing; `Files.createDirectories` would be safer. |
| **`Files.deleteIfExists`** | Removes files/directories. | Does not recurse; will not delete non‑empty directories. |

### 2.3 Architecture  

The class is a *storage layer* that sits behind the `ContentAssetsManager` abstraction. It delegates all persistence logic to the local file system, whereas other implementations (e.g., an Infinispan based one) would persist in-memory. The current implementation is tightly coupled to file‑system semantics and does not expose any abstractions for testing or alternative back‑ends.

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Returns | Side Effects | Comments |
|--------|---------|------------|---------|--------------|----------|
| `getInstance()` | Singleton accessor | none | `CmsStaticContentFileManagerImpl` | Instantiates the instance if null | Not thread‑safe. |
| `addFile(merchantStoreCode, folderPath, inputStaticContentData)` | Store a single file | merchant id, optional folder, file metadata | void | Copies file to disk | `folderPath` unused; potential path traversal. |
| `addFiles(merchantStoreCode, folderPath, list)` | Store many files | merchant id, optional folder, list of file metadata | void | Copies each file | `folderPath` unused. |
| `getFile(...)` | Retrieve a single file | merchant id, optional folder, type, name | `OutputContentFile` | N/A | Throws `ServiceException("Not implemented")`. |
| `getFiles(...)` | Retrieve multiple files | merchant id, optional folder, type | `List<OutputContentFile>` | N/A | Throws `ServiceException("Not implemented")`. |
| `removeFile(merchantStoreCode, type, fileName, folderPath)` | Delete one file | merchant id, type, file name, optional folder | void | Deletes file | `folderPath` unused. |
| `removeFiles(merchantStoreCode, folderPath)` | Delete all files for merchant | merchant id, optional folder | void | Deletes merchant directory (non‑recursive) | Should use recursive delete. |
| `getFileNames(merchantStoreCode, folderPath, staticContentType)` | List file names under a type | merchant id, optional folder, type | `List<String>` | N/A | Filters image files by MIME type. |
| `setRootName(String)` / `getRootName()` | Configure root directory name | root | void / String | N/A | Overrides default “static”. |
| `buildRootPath()` | Helper to build base path | none | `String` | N/A | Returns `rootName/ROOT_CONTAINER/` |
| `createDirectoryIfNorExist(Path)` | Ensures directory exists | path | void | Creates directory if missing | `createDirectory` will fail if parent missing. |
| `getCacheManager()` / `setCacheManager(LocalCacheManagerImpl)` | Accessors for cache manager | none / manager | manager / void | N/A | Allows injection of cache manager. |
| `addFolder(...)` | Create a sub‑folder | merchant id, folder name, optional folder path | void | Creates directory | Uses `buildMerchantPath`. |
| `removeFolder(...)` | Delete a sub‑folder | merchant id, folder name, optional folder path | void | Deletes directory | Non‑recursive. |
| `listFolders(...)` | List sub‑folders | merchant id, optional folder path | `List<String>` | N/A | Not implemented. |
| `getCmsManager()` | Access to CMS manager | none | `CMSManager` | N/A | Not implemented. |

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `java.nio.file.*` | Standard JDK | File system operations. |
| `org.slf4j.*` | Third‑party | Logging. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | String utilities. |
| `com.salesmanager.core.business.constants.Constants` | In‑house | Constants (e.g., `SLASH`). |
| `com.salesmanager.core.business.exception.ServiceException` | In‑house | Application‑level exception. |
| `com.salesmanager.core.business.modules.cms.content.ContentAssetsManager` | In‑house | Interface contract. |
| `com.salesmanager.core.business.modules.cms.impl.CMSManager` | In‑house | CMS configuration provider. |
| `com.salesmanager.core.business.modules.cms.impl.LocalCacheManagerImpl` | In‑house | Cache manager; cast to `CMSManager` in `init()`. |
| `com.salesmanager.core.model.content.*` | In‑house | DTOs for content files. |

All dependencies are either standard JDK or project‑specific. No external database or network libraries are used.

---

## 5. Additional Notes & Recommendations  

### 5.1 Thread Safety & Singleton

* `fileManager` is a plain static field; `getInstance()` is **not** synchronized or double‑checked.  
* In a multi‑threaded environment, multiple instances may be created concurrently, leading to inconsistent state.  
* **Fix**: Use an `enum` singleton, or make `fileManager` `volatile` and synchronize the initialization block.

### 5.2 `@PostConstruct` and Dependency Injection

* The class expects a container to inject `cacheManager` and call `init()`.  
* When the singleton is created via `new CmsStaticContentFileManagerImpl()`, `init()` will *not* run, leaving `rootName` at its default value.  
* **Fix**: Either remove the singleton pattern and rely on Spring to manage the bean, or manually invoke `init()` after setting the cache manager.

### 5.3 Path Handling

* Path construction uses manual string concatenation.  
* **Issue**: If `merchantStoreCode`, `fileName`, or `folderPath` contain `".."` or leading slashes, directory traversal attacks become possible.  
* **Fix**: Normalize paths via `Path.normalize()` and validate against forbidden patterns. Use `Path.resolve()` instead of `StringBuilder`.

### 5.4 Directory Creation

* `createDirectoryIfNorExist` uses `Files.createDirectory`, which fails if any parent directories are missing.  
* In `addFile`/`addFiles`, the root directory (`buildRootPath`) is created first, but intermediate directories (e.g., merchant code) may be missing if called concurrently.  
* **Fix**: Replace with `Files.createDirectories(path)` which ensures all parent directories are created.

### 5.5 Deletion Logic

* `removeFiles` and `removeFolder` use `Files.deleteIfExists`, which does not delete non‑empty directories.  
* **Issue**: Deleting a merchant folder that contains sub‑folders or files will silently fail.  
* **Fix**: Implement recursive delete (e.g., using `Files.walkFileTree`) or rely on a library like Apache Commons `FileUtils.deleteDirectory`.

### 5.6 Unused Parameters & Dead Code

* All methods accept an `Optional<String> folderPath` parameter, but it is never used.  
* This suggests incomplete refactoring or future plans that never materialized.  
* **Fix**: Remove the parameter or implement the intended folder handling.

### 5.7 Unimplemented Methods

* `getFile`, `getFiles`, `listFolders`, `getCmsManager` all throw a `ServiceException` with a “Not implemented” message.  
* This is acceptable as a placeholder but should be clearly documented or replaced with `UnsupportedOperationException` to signal that the operation is not supported by the local implementation.

### 5.8 Logging

* Logging is present, but the messages are fairly generic.  
* For debugging, include more context such as the full path, merchant ID, or the number of files processed.

### 5.9 Exception Handling

* All file operations are wrapped in a generic `catch (Exception)` block.  
* This swallows specific exceptions (e.g., `IOException` vs. `SecurityException`).  
* **Fix**: Catch more specific exceptions or re‑throw the original with context.

### 5.10 Performance & Scalability

* Copying files via `Files.copy` is efficient, but the code does not perform any deduplication or checksum validation.  
* If the same file is uploaded multiple times, it will be overwritten, potentially losing previous versions.  
* **Consider**: Add a policy for versioning or deduplication.

### 5.11 Testing & Mockability

* The heavy reliance on static paths and the singleton pattern makes unit testing difficult.  
* **Fix**: Inject a `PathResolver` or `FileSystemProvider` abstraction to allow mocking of file operations.

---

### Suggested Refactor Sketch

```java
@Service
public class LocalStaticContentFileManager implements ContentAssetsManager {

    private final Path basePath;
    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    public LocalStaticContentFileManager(CMSManager cmsManager) {
        this.basePath = Paths.get(cmsManager.getRootName(), "files");
        createIfNotExists(basePath);
    }

    @Override
    public void addFile(String merchantCode, Optional<String> folderPath,
                        InputContentFile file) throws ServiceException {
        Path target = resolveTargetPath(merchantCode, folderPath, file);
        createIfNotExists(target.getParent());
        try (InputStream in = file.getFile()) {
            Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new ServiceException("Failed to add file: " + target, e);
        }
    }

    // other methods similarly refactored...
}
```

* Uses constructor injection and Spring’s `@Service`.  
* `createIfNotExists` uses `Files.createDirectories`.  
* `resolveTargetPath` centralises path building and validation.  
* Eliminates the static singleton and `@PostConstruct` pitfalls.

---

**Bottom line:**  
The current implementation fulfills its basic role but contains several design weaknesses: a non‑thread‑safe singleton, incomplete dependency injection handling, unsafe path construction, non‑recursive deletes, unused parameters, and hard‑coded behavior for unimplemented methods. Refactoring towards a Spring‑managed bean with proper path validation, recursive cleanup, and clear separation of concerns would greatly improve maintainability, testability, and security.

## Code Critique



## Code Preview

```java
/**
 * 
 */
package com.salesmanager.core.business.modules.cms.content.local;

import java.io.IOException;
import java.net.URLConnection;
import java.nio.file.DirectoryStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import javax.annotation.PostConstruct;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.ContentAssetsManager;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.business.modules.cms.impl.LocalCacheManagerImpl;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * Manages - Images - Files (js, pdf, css...) on a local web server
 * 
 * @author Carl Samson
 * @since 1.0.3
 *
 */
public class CmsStaticContentFileManagerImpl implements ContentAssetsManager {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private static final Logger LOGGER = LoggerFactory.getLogger(CmsStaticContentFileManagerImpl.class);
	private static CmsStaticContentFileManagerImpl fileManager = null;
	private static final String ROOT_NAME = "static";

	private static final String ROOT_CONTAINER = "files";

	private String rootName = ROOT_NAME;

	private LocalCacheManagerImpl cacheManager;

	@PostConstruct
	void init() {

		this.rootName = ((CMSManager) cacheManager).getRootName();
		LOGGER.info("init " + getClass().getName() + " setting root" + this.rootName);

	}

	public static CmsStaticContentFileManagerImpl getInstance() {

		if (fileManager == null) {
			fileManager = new CmsStaticContentFileManagerImpl();
		}

		return fileManager;

	}

	/**
	 * <p>
	 * Method to add static content data for given merchant.Static content data
	 * can be of following type
	 * 
	 * <pre>
	 * 1. CSS and JS files
	 * 2. Digital Data like audio or video
	 * </pre>
	 * </p>
	 * <p>
	 * Merchant store code will be used to create cache node where merchant data
	 * will be stored,input data will contain name, file as well type of data
	 * being stored.
	 * 
	 * @see FileContentType
	 *      </p>
	 * 
	 * @param merchantStoreCode
	 *            merchant store for whom data is being stored
	 * @param inputStaticContentData
	 *            data object being stored
	 * @throws ServiceException
	 * 
	 */
	@Override
	public void addFile(final String merchantStoreCode, Optional<String> folderPath,
			final InputContentFile inputStaticContentData) throws ServiceException {
		/*
		 * if ( cacheManager.getTreeCache() == null ) { LOGGER.error(
		 * "Unable to find cacheManager.getTreeCache() in Infinispan.." ); throw
		 * new ServiceException(
		 * "CmsStaticContentFileManagerInfinispanImpl has a null cacheManager.getTreeCache()"
		 * ); }
		 */
		try {

			// base path
			String rootPath = this.buildRootPath();
			Path confDir = Paths.get(rootPath);
			this.createDirectoryIfNorExist(confDir);

			// node path
			StringBuilder nodePath = new StringBuilder();
			nodePath.append(rootPath).append(merchantStoreCode);
			Path merchantPath = Paths.get(nodePath.toString());
			this.createDirectoryIfNorExist(merchantPath);

			// file path
			nodePath.append(Constants.SLASH).append(inputStaticContentData.getFileContentType())
					.append(Constants.SLASH);
			Path dirPath = Paths.get(nodePath.toString());
			this.createDirectoryIfNorExist(dirPath);

			// folder path

			// file creation
			nodePath.append(inputStaticContentData.getFileName());

			Path path = Paths.get(nodePath.toString());

			// file creation
			// nodePath.append(Constants.SLASH).append(contentImage.getFileName());

			// Path path = Paths.get(nodePath.toString());

			Files.copy(inputStaticContentData.getFile(), path, StandardCopyOption.REPLACE_EXISTING);

			// String nodePath = this.getNodePath(merchantStoreCode,
			// inputStaticContentData.getFileContentType());

			// final Node<String, Object> merchantNode = this.getNode(nodePath);

			// merchantNode.put(inputStaticContentData.getFileName(),
			// IOUtils.toByteArray(
			// inputStaticContentData.getFile() ));

			LOGGER.info("Content data added successfully.");
		} catch (final Exception e) {
			LOGGER.error("Error while saving static content data", e);
			throw new ServiceException(e);

		}

	}

	/**
	 * <p>
	 * Method to add files for given store.Files will be stored in Infinispan
	 * and will be retrieved based on the storeID. Following steps will be
	 * performed to store static content files
	 * </p>
	 * <li>Merchant Node will be retrieved from the cacheTree if it exists else
	 * new node will be created.</li>
	 * <li>Files will be stored in StaticContentCacheAttribute , which
	 * eventually will be stored in Infinispan</li>
	 * 
	 * @param merchantStoreCode
	 *            Merchant store for which files are getting stored in
	 *            Infinispan.
	 * @param inputStaticContentDataList
	 *            input static content file list which will get
	 *            {@link InputContentImage} stored
	 * @throws ServiceException
	 *             if content file storing process fail.
	 * @see InputStaticContentData
	 * @see StaticContentCacheAttribute
	 */
	@Override
	public void addFiles(final String merchantStoreCode, Optional<String> folderPath,
			final List<InputContentFile> inputStaticContentDataList) throws ServiceException {
		/*
		 * if ( cacheManager.getTreeCache() == null ) { LOGGER.error(
		 * "Unable to find cacheManager.getTreeCache() in Infinispan.." ); throw
		 * new ServiceException(
		 * "CmsStaticContentFileManagerInfinispanImpl has a null cacheManager.getTreeCache()"
		 * ); }
		 */
		try {

			// base path
			String rootPath = this.buildRootPath();
			Path confDir = Paths.get(rootPath);
			this.createDirectoryIfNorExist(confDir);

			// node path
			StringBuilder nodePath = new StringBuilder();
			nodePath.append(rootPath).append(merchantStoreCode);
			Path merchantPath = Paths.get(nodePath.toString());
			this.createDirectoryIfNorExist(merchantPath);

			for (final InputContentFile inputStaticContentData : inputStaticContentDataList) {

				// file path
				nodePath.append(Constants.SLASH).append(inputStaticContentData.getFileContentType())
						.append(Constants.SLASH);
				Path dirPath = Paths.get(nodePath.toString());
				this.createDirectoryIfNorExist(dirPath);

				// file creation
				nodePath.append(Constants.SLASH).append(inputStaticContentData.getFileName());

				Path path = Paths.get(nodePath.toString());

				Files.copy(inputStaticContentData.getFile(), path, StandardCopyOption.REPLACE_EXISTING);

				// String nodePath = this.getNodePath(merchantStoreCode,
				// inputStaticContentData.getFileContentType());

				// final Node<String, Object> merchantNode =
				// this.getNode(nodePath);
				// merchantNode.put(inputStaticContentData.getFileName(),
				// IOUtils.toByteArray(
				// inputStaticContentData.getFile() ));

			}

			LOGGER.info("Total {} files added successfully.", inputStaticContentDataList.size());

		} catch (final Exception e) {
			LOGGER.error("Error while saving content image", e);
			throw new ServiceException(e);

		}
	}

	/**
	 * Method to return static data for given Merchant store based on the file
	 * name. Content data will be searched in underlying Infinispan cache tree
	 * and {@link OutputStaticContentData} will be returned on finding an
	 * associated file. In case of no file, null be returned.
	 * 
	 * @param store
	 *            Merchant store
	 * @param contentFileName
	 *            name of file being requested
	 * @return {@link OutputStaticContentData}
	 * @throws ServiceException
	 */
	@Override
	public OutputContentFile getFile(final String merchantStoreCode, Optional<String> folderPath,
			final FileContentType fileContentType, final String contentFileName) throws ServiceException {

		throw new ServiceException("Not implemented for httpd image manager");

	}

	@Override
	public List<OutputContentFile> getFiles(final String merchantStoreCode, Optional<String> folderPath,
			final FileContentType staticContentType) throws ServiceException {

		throw new ServiceException("Not implemented for httpd image manager");

	}

	@Override
	public void removeFile(final String merchantStoreCode, final FileContentType staticContentType,
			final String fileName, Optional<String> folderPath) throws ServiceException {

		try {

			StringBuilder merchantPath = new StringBuilder();
			merchantPath.append(buildRootPath()).append(Constants.SLASH).append(merchantStoreCode)
					.append(Constants.SLASH).append(staticContentType).append(Constants.SLASH).append(fileName);

			Path path = Paths.get(merchantPath.toString());

			Files.deleteIfExists(path);

		} catch (final Exception e) {
			LOGGER.error("Error while deleting files for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	/**
	 * Removes the data in a given merchant node
	 */
	@Override
	public void removeFiles(final String merchantStoreCode, Optional<String> folderPath) throws ServiceException {

		LOGGER.debug("Removing all images for {} merchant ", merchantStoreCode);

		try {

			StringBuilder merchantPath = new StringBuilder();
			merchantPath.append(buildRootPath()).append(Constants.SLASH).append(merchantStoreCode);

			Path path = Paths.get(merchantPath.toString());

			Files.deleteIfExists(path);

		} catch (final Exception e) {
			LOGGER.error("Error while deleting content image for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	/**
	 * Queries the CMS to retrieve all static content files. Only the name of
	 * the file will be returned to the client
	 * 
	 * @param merchantStoreCode
	 * @return
	 * @throws ServiceException
	 */
	@Override
	public List<String> getFileNames(final String merchantStoreCode, Optional<String> folderPath,
			final FileContentType staticContentType) throws ServiceException {

		try {

			StringBuilder merchantPath = new StringBuilder();
			merchantPath.append(buildRootPath()).append(merchantStoreCode).append(Constants.SLASH)
					.append(staticContentType);

			Path path = Paths.get(merchantPath.toString());

			List<String> fileNames = null;

			if (Files.exists(path)) {

				fileNames = new ArrayList<String>();
				try (DirectoryStream<Path> directoryStream = Files.newDirectoryStream(path)) {
					for (Path dirPath : directoryStream) {

						String fileName = dirPath.getFileName().toString();

						if (staticContentType.name().equals(FileContentType.IMAGE.name())) {
							// File f = new File(fileName);
							String mimetype = URLConnection.guessContentTypeFromName(fileName);
							// String mimetype= new
							// MimetypesFileTypeMap().getContentType(f);
							if (!StringUtils.isBlank(mimetype)) {
								String type = mimetype.split("/")[0];
								if (type.equals("image")) {
									fileNames.add(fileName);
								}
							}
							// fileNames.add(fileName);

						} else {
							fileNames.add(fileName);
						}

					}
				}

				return fileNames;
			}
		} catch (final Exception e) {
			LOGGER.error("Error while fetching file for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}
		return new ArrayList<>();
	}

	public void setRootName(String rootName) {
		this.rootName = rootName;
	}

	public String getRootName() {
		return rootName;
	}

	private String buildRootPath() {
		return new StringBuilder().append(getRootName()).append(Constants.SLASH).append(ROOT_CONTAINER)
				.append(Constants.SLASH).toString();

	}

	private void createDirectoryIfNorExist(Path path) throws IOException {

		if (Files.notExists(path)) {
			Files.createDirectory(path);
		}
	}

	public LocalCacheManagerImpl getCacheManager() {
		return cacheManager;
	}

	public void setCacheManager(LocalCacheManagerImpl cacheManager) {
		this.cacheManager = cacheManager;
	}

	@Override
	public void addFolder(String merchantStoreCode, String folderName, Optional<String> folderPath)
			throws ServiceException {


		try {

			Path merchantPath = this.buildMerchantPath(merchantStoreCode);
			
			StringBuilder nodePath = new StringBuilder();
			if(folderPath.isPresent()) {
				nodePath
				.append(merchantPath.toString())
				.append(Constants.SLASH).append(folderPath.get()).append(Constants.SLASH);
			}
			// add folder
			nodePath.append(folderName);
			
			Path dirPath = Paths.get(nodePath.toString());
			this.createDirectoryIfNorExist(dirPath);

		} catch (IOException e) {
			LOGGER.error("Error while creating fiolder for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}
	
	private Path buildMerchantPath(String merchantCode) throws IOException {
		
		String rootPath = this.buildRootPath();
		Path confDir = Paths.get(rootPath);
		
		// node path
		StringBuilder nodePath = new StringBuilder();
		nodePath
		.append(confDir.toString())
		.append(rootPath).append(merchantCode);
		Path merchantPath = Paths.get(nodePath.toString());
		this.createDirectoryIfNorExist(merchantPath);
		
		return merchantPath;
		
	}

	@Override
	public void removeFolder(String merchantStoreCode, String folderName, Optional<String> folderPath)
			throws ServiceException {

		//String rootPath = this.buildRootPath();
		try {
			
			Path merchantPath = this.buildMerchantPath(merchantStoreCode);
			StringBuilder nodePath = new StringBuilder();
			nodePath.append(merchantPath.toString()).append(Constants.SLASH);
			if(folderPath.isPresent()) {
				nodePath.append(folderPath.get()).append(Constants.SLASH);
			}
			
			nodePath.append(folderName);
			
			Path longPath = Paths.get(nodePath.toString());
			
			if (Files.exists(longPath)) {
				Files.delete(longPath);
			}
			
		} catch (IOException e) {
			LOGGER.error("Error while creating fiolder for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	@Override
	public List<String> listFolders(String merchantStoreCode, Optional<String> folderPath) throws ServiceException {
		// TODO Auto-generated method stub
		return null;
	}

	@Override
	public CMSManager getCmsManager() {
	  return null;
	}

}



```
