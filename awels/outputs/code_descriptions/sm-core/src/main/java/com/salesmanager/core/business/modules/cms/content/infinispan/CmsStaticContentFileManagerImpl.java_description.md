# CmsStaticContentFileManagerImpl.java

## Review

## 1. Summary  

**Purpose**  
`CmsStaticContentFileManagerImpl` is a lightweight Infinispan‑based persistence layer that stores and retrieves static assets (images, CSS/JS, audio, etc.) per merchant.  
It implements the `ContentAssetsManager` interface and exposes CRUD operations for single files, bulk uploads, folder management, and file listing.

**Key Components**  

| Component | Role |
|-----------|------|
| `CacheManager` | Provides access to the Infinispan `TreeCache` used for the file store. |
| `Node<String,Object>` | Infinispan tree node that holds file data as byte arrays keyed by the file name. |
| `InputContentFile` / `OutputContentFile` | DTOs that carry file streams and metadata. |
| `FileContentType` | Enum that scopes the files (e.g. IMAGE, JS, PDF). |
| `CmsStaticContentFileManagerImpl` | The manager implementation itself. |

**Design notes**  
- A **singleton pattern** (`getInstance`) is used, but it is **not thread‑safe** and is redundant if the class is instantiated by a dependency injection framework.  
- The class is annotated with `@PostConstruct`, implying that a container (Spring, CDI, etc.) is expected to inject the `CacheManager`.  
- It relies heavily on **Infinispan TreeCache** and the `Fqn` utility to address nodes.  
- The code contains many *unused* or *incomplete* methods (`removeFolder`, `listFolders`, `getFolder`, `getCmsManager`).

---

## 2. Detailed Description  

### Initialization  

1. **`@PostConstruct init()`**  
   - Pulls the root name (`static-merchant-…`) from the injected `CacheManager`.  
   - Logs the root name; if `cacheManager` is not injected, the class will throw a `NullPointerException` later.

2. **Singleton (`getInstance()`)**  
   - Lazily creates a single instance.  
   - No synchronization ⇒ race condition if two threads call `getInstance()` concurrently.

### Core CRUD Flow  

| Operation | Flow | Notes |
|-----------|------|-------|
| **Add file (`addFile`)** | - Build node path: `merchantStoreCode + "/" + contentType`. <br> - Retrieve or create the node via `getNode()`. <br> - Convert the input stream to a byte array and put it into the node keyed by file name. | Entire file is read into memory (`IOUtils.toByteArray`). May cause OOM for large files. |
| **Add files (`addFiles`)** | Loops over the list and reuses the same logic as `addFile`. | Same memory concerns; no batch commit. |
| **Get file (`getFile`)** | - Resolve node as above. <br> - Fetch the byte array. <br> - Wrap it into a `ByteArrayInputStream` → `ByteArrayOutputStream`. <br> - Fill `OutputContentFile` with MIME type derived from `URLConnection.getFileNameMap()`. | Returns `null` if not found. `null` is acceptable but callers must handle it. |
| **Get files (`getFiles`)** | Enumerates all keys in the node, creates an `OutputContentFile` for each, and returns a list. | Potentially expensive if many files. |
| **Remove file (`removeFile`)** | Calls `node.remove(fileName)`. | No check for existence; silent failure if key missing. |
| **Remove all files (`removeFiles`)** | Builds the path for the merchant root and removes the node entirely. | Deleting the root node also removes any sub‑folders or metadata that may exist there. |
| **Folder management (`addFolder`)** | Builds a path that includes an optional user‑provided sub‑path, then creates the folder node in the tree. | Implementation is fragile: it uses `FileContentType.IMAGE` as the base, ignores the `path` variable, and never checks if the folder already exists. |

### Error Handling  

All public methods wrap any exception in a `ServiceException` and log it.  
Generic `Exception` catching hides the precise cause and makes debugging harder.  

### Dependencies & Assumptions  

| Library | Purpose | Comments |
|---------|---------|----------|
| Infinispan (`org.infinispan.tree.*`) | Provides the distributed tree cache and node abstraction. | Requires a running Infinispan server or embedded instance. |
| Apache Commons IO (`IOUtils`) | Stream → byte[] conversion and copying. | Ok, but can be replaced by Java NIO for large files. |
| SLF4J | Logging. | Good practice. |
| `java.net.URLConnection` | MIME type detection. | Might return `null`; fallback logic missing. |

The code assumes that the `CacheManager` is correctly configured and that the `TreeCache` is available. If `cacheManager.getTreeCache()` returns `null`, the methods throw `ServiceException`.  

### Architecture & Design Choices  

- **TreeCache** provides hierarchical storage that maps nicely to the merchant → content type → file name structure.  
- The **node path** always starts with a merchant‑specific root (`rootName + storeCode`).  
- The class acts as a thin façade over Infinispan, with no business logic or transaction support.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `stopFileManager()` | Stops the underlying Infinispan manager. | none | none | Calls `cacheManager.getManager().stop()`. |
| `init()` | @PostConstruct initializer; sets root name. | none | none | Logs initialization. |
| `getInstance()` | Singleton factory. | none | instance | Creates a new instance if `fileManager` is `null`. |
| `addFile(String, Optional<String>, InputContentFile)` | Stores a single file. | merchant code, optional path (unused), file DTO | none | Writes byte array to node. |
| `addFiles(String, Optional<String>, List<InputContentFile>)` | Bulk store. | merchant code, optional path (unused), list | none | Writes each file to node. |
| `getFile(String, Optional<String>, FileContentType, String)` | Retrieve single file. | merchant code, optional path (unused), type, name | `OutputContentFile` | Reads node value. |
| `getFiles(String, Optional<String>, FileContentType)` | Retrieve all files of a type. | merchant code, optional path (unused), type | List of `OutputContentFile` | Enumerates node keys. |
| `removeFile(String, FileContentType, String, Optional<String>)` | Delete single file. | merchant code, type, name, optional path (unused) | none | Calls `node.remove()`. |
| `removeFiles(String, Optional<String>)` | Delete all files of a merchant. | merchant code, optional path (unused) | none | Removes merchant node. |
| `getNode(String)` | Internal helper that resolves or creates a node. | node path (relative to root) | `Node<String,Object>` | May create a new child node. |
| `getNodePath(String, FileContentType)` | Builds the relative node path. | store code, type | `String` | `"storeCode/type"` |
| `getFolder(String, String)` | (Unimplemented) Intended to return folder path. | store, folder | `String` | `null` |
| `getCacheManager()` / `setCacheManager(CacheManager)` | Dependency injection getters/setters. | none / manager | manager / none | |
| `getFileNames(String, Optional<String>, FileContentType)` | List only the names of files. | merchant code, optional path (unused), type | `List<String>` | |
| `addFolder(String, String, Optional<String>)` | Creates a folder node. | merchant code, folder name, optional path | none | Creates Fqn nodes. |
| `removeFolder(String, String, Optional<String>)` | (Unimplemented) Intended to delete a folder. | merchant code, folder name, optional path | none | |
| `listFolders(String, Optional<String>)` | (Unimplemented) Intended to list folders. | merchant code, optional path | `List<String>` | |
| `getCmsManager()` | (Unimplemented) Should return a `CMSManager`. | none | `null` | |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.infinispan.tree.*` | Third‑party (Infinispan) | Core to the storage logic. |
| `org.apache.commons.io.IOUtils` | Third‑party | Stream handling; may be replaced by Java NIO. |
| `org.slf4j.*` | Third‑party | Logging. |
| `javax.annotation.PostConstruct` | Standard | CDI/Spring annotation. |
| `java.net.URLConnection` | Standard | MIME type lookup. |
| `java.util.Optional` | Standard | Used for optional path, but never utilized. |

No native platform dependencies, but the code requires a functioning Infinispan instance and the `CacheManager` to be properly configured.

---

## 5. Additional Notes & Recommendations  

### 5.1 Thread Safety  
- The singleton pattern is **not synchronized**. In a multithreaded environment, two threads may create separate instances.  
- Consider using an enum singleton or rely on the DI container to provide a single bean.

### 5.2 Memory & Performance  
- Reading entire files into a byte array (`IOUtils.toByteArray`) can cause memory pressure.  
- For large files, stream the data directly into a `Blob` or use Infinispan’s `byte[]` support with streaming APIs.  
- `getFiles` enumerates all keys; if a merchant has many assets, this can become expensive. Pagination or a separate index may help.

### 5.3 Null/Optional Handling  
- The `Optional<String> path` parameter is currently unused in all CRUD methods except `addFolder`.  
- Either remove it or integrate it into the node path logic to allow sub‑folder support.  
- `getFile` returns `null` on missing file; callers should be prepared for this, or use `Optional<OutputContentFile>`.

### 5.4 Error Handling  
- Swallowing generic `Exception` hides specific issues. Catch only the exceptions you expect (e.g., `IOException`, `NullPointerException`).  
- Provide more context in logs: include the merchant code, file name, and the exact exception message.

### 5.5 Method Implementations  
- **`getFolder`** returns `null`; if intended, document why.  
- **`removeFolder`** and **`listFolders`** are stubbed out; implement or remove them.  
- **`getCmsManager`** returns `null`; either implement or remove from interface.

### 5.6 Coding Style & Cleanup  
- The class has a `serialVersionUID` but does not implement `Serializable`. Remove the field.  
- Use generics more precisely: `Node<String, byte[]>` instead of raw `Object`.  
- Replace string concatenation with `StringBuilder` only when performance matters; otherwise use `String.format`.  
- Remove commented‑out code and unused imports.

### 5.7 API Design  
- Consider returning a DTO that contains the file size and timestamp.  
- Provide a `saveFile` method that accepts a `File` object or `Path` to avoid manual stream conversion.  

### 5.8 Security  
- No access control checks – anyone with access to the manager can read or delete any merchant’s files.  
- Add authorization checks based on the `merchantStoreCode` or user context.

---

### Summary of Key Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| Non‑thread‑safe singleton | Race condition, multiple instances | Use DI container or synchronize |
| Unused `Optional<String> path` | Confusing API | Remove or fully integrate |
| In-memory file read | OOM for large files | Stream to cache, use `Blob` |
| Unimplemented methods | API contract violations | Implement or remove |
| Generic exception catching | Hard to debug | Catch specific exceptions |
| `serialVersionUID` unused | Unnecessary code | Remove |
| Lack of access control | Security risk | Add authorization layer |

Implementing these improvements will make the class more robust, maintainable, and production‑ready.

## Code Critique



## Code Preview

```java
/**
 * 
 */
package com.salesmanager.core.business.modules.cms.content.infinispan;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.FileNameMap;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Optional;

import javax.annotation.PostConstruct;
import org.apache.commons.io.IOUtils;
import org.infinispan.tree.Fqn;
import org.infinispan.tree.Node;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.ContentAssetsManager;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.business.modules.cms.impl.CacheManager;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * Manages - Images - Files (js, pdf, css...) on infinispan
 * 
 * @author Umesh Awasthi
 * @since 1.2
 *
 */
public class CmsStaticContentFileManagerImpl
		// implements FilePut,FileGet,FileRemove
		implements ContentAssetsManager {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private static final Logger LOGGER = LoggerFactory.getLogger(CmsStaticContentFileManagerImpl.class);
	private static CmsStaticContentFileManagerImpl fileManager = null;
	private static final String ROOT_NAME = "static-merchant-";

	private String rootName = ROOT_NAME;

	private CacheManager cacheManager;

	public void stopFileManager() {

		try {
			cacheManager.getManager().stop();
			LOGGER.info("Stopping CMS");
		} catch (final Exception e) {
			LOGGER.error("Error while stopping CmsStaticContentFileManager", e);
		}
	}

	@PostConstruct
	void init() {

		this.rootName = cacheManager.getRootName();
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
	public void addFile(final String merchantStoreCode, Optional<String>path, final InputContentFile inputStaticContentData)
			throws ServiceException {
		if (cacheManager.getTreeCache() == null) {
			LOGGER.error("Unable to find cacheManager.getTreeCache() in Infinispan..");
			throw new ServiceException(
					"CmsStaticContentFileManagerInfinispanImpl has a null cacheManager.getTreeCache()");
		}
		try {

			String nodePath = this.getNodePath(merchantStoreCode, inputStaticContentData.getFileContentType());

			final Node<String, Object> merchantNode = this.getNode(nodePath);

			merchantNode.put(inputStaticContentData.getFileName(),
					IOUtils.toByteArray(inputStaticContentData.getFile()));

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
	public void addFiles(final String merchantStoreCode, Optional<String> path, final List<InputContentFile> inputStaticContentDataList)
			throws ServiceException {
		if (cacheManager.getTreeCache() == null) {
			LOGGER.error("Unable to find cacheManager.getTreeCache() in Infinispan..");
			throw new ServiceException(
					"CmsStaticContentFileManagerInfinispanImpl has a null cacheManager.getTreeCache()");
		}
		try {

			for (final InputContentFile inputStaticContentData : inputStaticContentDataList) {

				String nodePath = this.getNodePath(merchantStoreCode, inputStaticContentData.getFileContentType());
				final Node<String, Object> merchantNode = this.getNode(nodePath);
				merchantNode.put(inputStaticContentData.getFileName(),
						IOUtils.toByteArray(inputStaticContentData.getFile()));

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
	public OutputContentFile getFile(final String merchantStoreCode, Optional<String> path, final FileContentType fileContentType,
			final String contentFileName) throws ServiceException {

		if (cacheManager.getTreeCache() == null) {
			throw new ServiceException("CmsStaticContentFileManagerInfinispan has a null cacheManager.getTreeCache()");
		}
		OutputContentFile outputStaticContentData = new OutputContentFile();
		InputStream input = null;
		try {

			String nodePath = this.getNodePath(merchantStoreCode, fileContentType);

			final Node<String, Object> merchantNode = this.getNode(nodePath);

			final byte[] fileBytes = (byte[]) merchantNode.get(contentFileName);

			if (fileBytes == null) {
				LOGGER.warn("file byte is null, no file found");
				return null;
			}

			input = new ByteArrayInputStream(fileBytes);

			final ByteArrayOutputStream output = new ByteArrayOutputStream();
			IOUtils.copy(input, output);

			outputStaticContentData.setFile(output);
			outputStaticContentData.setMimeType(URLConnection.getFileNameMap().getContentTypeFor(contentFileName));
			outputStaticContentData.setFileName(contentFileName);
			outputStaticContentData.setFileContentType(fileContentType);

		} catch (final Exception e) {
			LOGGER.error("Error while fetching file for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}
		return outputStaticContentData;
	}

	@Override
	public List<OutputContentFile> getFiles(final String merchantStoreCode, Optional<String> path, final FileContentType staticContentType)
			throws ServiceException {

		if (cacheManager.getTreeCache() == null) {
			throw new ServiceException("CmsStaticContentFileManagerInfinispan has a null cacheManager.getTreeCache()");
		}
		List<OutputContentFile> images = new ArrayList<OutputContentFile>();
		try {

			FileNameMap fileNameMap = URLConnection.getFileNameMap();
			String nodePath = this.getNodePath(merchantStoreCode, staticContentType);

			final Node<String, Object> merchantNode = this.getNode(nodePath);

			for (String key : merchantNode.getKeys()) {

				byte[] imageBytes = (byte[]) merchantNode.get(key);

				OutputContentFile contentImage = new OutputContentFile();

				InputStream input = new ByteArrayInputStream(imageBytes);
				ByteArrayOutputStream output = new ByteArrayOutputStream();
				IOUtils.copy(input, output);

				String contentType = fileNameMap.getContentTypeFor(key);

				contentImage.setFile(output);
				contentImage.setMimeType(contentType);
				contentImage.setFileName(key);

				images.add(contentImage);

			}

		} catch (final Exception e) {
			LOGGER.error("Error while fetching file for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

		return images;

	}

	@Override
	public void removeFile(final String merchantStoreCode, final FileContentType staticContentType,
			final String fileName, Optional<String> path) throws ServiceException {

		if (cacheManager.getTreeCache() == null) {
			throw new ServiceException("CmsStaticContentFileManagerInfinispan has a null cacheManager.getTreeCache()");
		}

		try {

			String nodePath = this.getNodePath(merchantStoreCode, staticContentType);
			final Node<String, Object> merchantNode = this.getNode(nodePath);

			merchantNode.remove(fileName);

		} catch (final Exception e) {
			LOGGER.error("Error while fetching file for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	/**
	 * Removes the data in a given merchant node
	 */
	@SuppressWarnings("unchecked")
	@Override
	public void removeFiles(final String merchantStoreCode, Optional<String> path) throws ServiceException {

		LOGGER.info("Removing all images for {} merchant ", merchantStoreCode);
		if (cacheManager.getTreeCache() == null) {
			LOGGER.error("Unable to find cacheManager.getTreeCache() in Infinispan..");
			throw new ServiceException("CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
		}

		try {

			final StringBuilder merchantPath = new StringBuilder();
			merchantPath.append(getRootName()).append(merchantStoreCode);
			cacheManager.getTreeCache().getRoot().remove(merchantPath.toString());

		} catch (final Exception e) {
			LOGGER.error("Error while deleting content image for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	@SuppressWarnings({ "unchecked" })
	private Node<String, Object> getNode(final String node) {
		LOGGER.debug("Fetching node for store {} from Infinispan", node);
		final StringBuilder merchantPath = new StringBuilder();
		merchantPath.append(getRootName()).append(node);

		Fqn contentFilesFqn = Fqn.fromString(merchantPath.toString());

		Node<String, Object> nd = cacheManager.getTreeCache().getRoot().getChild(contentFilesFqn);

		if (nd == null) {

			cacheManager.getTreeCache().getRoot().addChild(contentFilesFqn);
			nd = cacheManager.getTreeCache().getRoot().getChild(contentFilesFqn);

		}

		return nd;

	}

	private String getNodePath(final String storeCode, final FileContentType contentType) {

		StringBuilder nodePath = new StringBuilder();
		nodePath.append(storeCode).append("/").append(contentType.name());

		return nodePath.toString();

	}
	
	
	/**
	 * Returns a folder path so it can be used as base node
	 * @param storeCode
	 * @param folder
	 * @return
	 */
	private String getFolder(final String storeCode, String folder) {

/*		StringBuilder nodePath = new StringBuilder();
		nodePath.append(storeCode).append("/").append(contentType.name());

		return nodePath.toString();*/
		
		
		return null;

	}

	public CacheManager getCacheManager() {
		return cacheManager;
	}

	public void setCacheManager(CacheManager cacheManager) {
		this.cacheManager = cacheManager;
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
	public List<String> getFileNames(final String merchantStoreCode, Optional<String> path, final FileContentType staticContentType)
			throws ServiceException {

		if (cacheManager.getTreeCache() == null) {
			throw new ServiceException("CmsStaticContentFileManagerInfinispan has a null cacheManager.getTreeCache()");
		}

		try {

			String nodePath = this.getNodePath(merchantStoreCode, staticContentType);
			final Node<String, Object> objectNode = this.getNode(nodePath);

			if (objectNode.getKeys().isEmpty()) {
				LOGGER.warn("Unable to find content attribute for given merchant");
				return Collections.emptyList();
			}
			return new ArrayList<String>(objectNode.getKeys());

		} catch (final Exception e) {
			LOGGER.error("Error while fetching file for {} merchant ", merchantStoreCode);
			throw new ServiceException(e);
		}

	}

	public void setRootName(String rootName) {
		this.rootName = rootName;
	}

	public String getRootName() {
		return rootName;
	}

	@SuppressWarnings("unchecked")
	@Override
	public void addFolder(String merchantStoreCode, String folderName, Optional<String> path) throws ServiceException {
		
		
		String nodePath = this.getNodePath(merchantStoreCode, FileContentType.IMAGE);
		
		StringBuilder appender = new StringBuilder();
		appender.append(nodePath).append(Constants.SLASH);

		path.ifPresent(appender::append);
		
		
		//Put logic in a method
		
		Fqn folderFqn = Fqn.fromString(appender.toString());

		Node<String, Object> nd = cacheManager.getTreeCache().getRoot().getChild(folderFqn);

		if (nd == null) {

			cacheManager.getTreeCache().getRoot().addChild(folderFqn);
			nd = cacheManager.getTreeCache().getRoot().getChild(folderFqn);

		}
		
		appender.append(Constants.SLASH).append(folderName);
		
		Fqn newFolderFqn = Fqn.fromString(appender.toString());
		cacheManager.getTreeCache().getRoot().addChild(newFolderFqn);

	}

	@Override
	public void removeFolder(String merchantStoreCode, String folderName, Optional<String> path) throws ServiceException {
		// TODO Auto-generated method stub

	}

	@Override
	public List<String> listFolders(String merchantStoreCode, Optional<String> path) throws ServiceException {
		// TODO Auto-generated method stub
		return null;
	}

	@Override
	public CMSManager getCmsManager() {
    	return null;
  	}

}



```
