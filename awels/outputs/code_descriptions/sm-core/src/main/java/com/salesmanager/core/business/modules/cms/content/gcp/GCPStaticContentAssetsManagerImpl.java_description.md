# GCPStaticContentAssetsManagerImpl.java

## Review

## 1. Summary  
**Purpose** – This class provides a concrete implementation of the `ContentAssetsManager` interface for a retailer that stores static assets in Google Cloud Storage (GCS).  
**Key components**  
| Component | Role |
|-----------|------|
| `GCPStaticContentAssetsManagerImpl` | Spring‐managed bean that orchestrates CRUD operations on GCS objects. |
| `cmsManager` | Injected `CMSManager` (currently unused; likely intended to provide configuration such as bucket names). |
| GCS SDK (`com.google.cloud.storage.*`) | Underlying client library used to communicate with Google Cloud Storage. |

**Notable design choices**  
* Uses the *Strategy* pattern – the implementation is swapped in via Spring’s component name `"gcpContentAssetsManager"`.  
* Leverages `Optional<String>` for the folder path, suggesting future support for hierarchical folders even though GCS is flat.  
* Minimal dependency on Spring (only component and autowiring).  

---

## 2. Detailed Description  
### Flow of execution  
1. **Initialization** – Spring creates an instance of `GCPStaticContentAssetsManagerImpl` and injects a `CMSManager`.  
2. **Runtime** – Every public method (`addFile`, `removeFile`, etc.) obtains a `Storage` client via `StorageOptions.getDefaultInstance().getService()` and then performs the desired operation on a GCS bucket.  
3. **Path handling** – All object names are prefixed by the result of `nodePath(merchantStoreCode, fileContentType)` (or a variant that includes the file name). This method is not shown in the snippet but is assumed to build a key such as `merchantCode/contentType/`.  
4. **Cleanup** – The class itself has no explicit cleanup logic; GCS clients are lightweight and managed by the SDK.  

### Assumptions & constraints  
* The bucket name is retrieved via a private helper `bucketName()`, which is also not shown.  
* GCS prefixes are treated as “folders”; actual folder deletion is performed by deleting a single object key (not a batch delete of all keys with that prefix).  
* `Optional<String> folderPath` is currently ignored – a placeholder for future hierarchical support.  
* `InputContentFile.getFile()` is a `java.io.InputStream`; the implementation assumes it supports `available()` and can be read into a byte array, which may not be safe for large streams.  

### Architecture & design choices  
* **Simplicity** – The code focuses on a narrow responsibility (CRUD for GCS) and keeps the surface area small.  
* **Extensibility** – By using the `ContentAssetsManager` interface, other storage back‑ends could be swapped in (e.g., AWS S3).  
* **Missing abstraction** – Helper methods (`bucketName()`, `nodePath()`, `getName()`, `isInsideSubFolder()`) are not defined in the provided snippet, making it harder to reason about correctness.  
* **Exception handling** – All operations catch generic `Exception` and wrap it in a `ServiceException`, which is acceptable but could be more granular.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects | Notes |
|--------|---------|--------|---------|--------------|-------|
| `getFile(...)` | Retrieve a single file from GCS. | `merchantStoreCode`, `folderPath`, `fileContentType`, `contentName` | `OutputContentFile` | None | Calls `blob.getContent()` and converts it to `OutputContentFile`. |
| `getFileNames(...)` | List file names for a given content type. | `merchantStoreCode`, `folderPath`, `fileContentType` | `List<String>` | None | Uses `bucket.list()` with `BlobField.NAME`. |
| `getFiles(...)` | Return all `OutputContentFile`s for a content type. | `merchantStoreCode`, `folderPath`, `fileContentType` | `List<OutputContentFile>` | None | Calls `getFileNames` + `getFile`. |
| `addFile(...)` | Upload a single file to GCS. | `merchantStoreCode`, `folderPath`, `InputContentFile` | `void` | Creates object in bucket. | Reads entire stream into memory – potential memory issue. |
| `addFiles(...)` | Upload multiple files. | `merchantStoreCode`, `folderPath`, `List<InputContentFile>` | `void` | Iteratively calls `addFile`. | Uses `CollectionUtils.isNotEmpty`. |
| `removeFile(...)` | Delete a single file from GCS. | `merchantStoreCode`, `staticContentType`, `fileName`, `folderPath` | `void` | Deletes object. | Does not verify deletion success. |
| `removeFiles(...)` | Delete a “folder” (prefix) from GCS. | `merchantStoreCode`, `folderPath` | `void` | Calls `storage.delete(bucketName, nodePath(...))`. | Only deletes the prefix object, not all objects under that prefix. |
| `addFolder(...)` / `removeFolder(...)` / `listFolders(...)` | Stub methods for folder support. | Varies | Varies | None | Not implemented. |
| `getCmsManager()` / `setCmsManager()` | Bean accessors. | None | `CMSManager` | None | `cmsManager` is currently unused. |

**Utility methods** (assumed but not shown)  
* `bucketName()` – retrieves the configured bucket name.  
* `nodePath(merchantStoreCode, fileContentType)` – builds the object key prefix.  
* `getName(blobName)` – extracts the file name from a full key.  
* `isInsideSubFolder(blobName)` – determines if a blob lies inside a sub‑folder.  

---

## 4. Dependencies  

| Library | Type | Purpose |
|---------|------|---------|
| Spring (`org.springframework.*`) | Third‑party | Dependency injection and component declaration. |
| Google Cloud Storage SDK (`com.google.cloud.storage.*`) | Third‑party | Interaction with GCS. |
| Apache Commons Lang (`org.apache.commons.lang3.*`) | Third‑party | Utility functions (`StringUtils`). |
| Apache Commons Collections (`org.apache.commons.collections4.*`) | Third‑party | Collection helpers (`CollectionUtils`). |
| SLF4J (`org.slf4j.*`) | Third‑party | Logging. |
| Custom (`com.salesmanager.*`) | Application | Domain models (`FileContentType`, `InputContentFile`, `OutputContentFile`) and exceptions (`ServiceException`). |

No platform‑specific APIs are used; all dependencies are standard Java / Maven artifacts.

---

## 5. Additional Notes  

### Edge‑cases & potential pitfalls  

| Scenario | Issue | Suggested Fix |
|----------|-------|---------------|
| Large file uploads | `InputContentFile.getFile().available()` may return 0; reading into a byte array can lead to `OutOfMemoryError`. | Use `ByteStreams.toByteArray(InputStream)` or stream directly to GCS with `storage.create(BlobInfo, InputStream)`. |
| Deleting “folders” | `storage.delete(bucketName, nodePath(...))` only deletes the object with that key, not all objects sharing the prefix. | Use `bucket.list(BlobListOption.prefix(...))` and batch delete all matching blobs. |
| Unused `folderPath` | Currently ignored; future callers might rely on it. | Either remove the parameter or implement prefix handling. |
| Unimplemented folder methods | Returning `null` or doing nothing can lead to `NullPointerException` downstream. | Throw `UnsupportedOperationException` or provide a proper implementation. |
| Missing helper methods | `bucketName()`, `nodePath()`, etc. are not defined, making the class incomplete. | Ensure these methods are implemented in this class or a superclass. |
| Logging granularity | Generic “Content add file” messages do not include context. | Include merchant store code or file name in log statements. |
| Exception handling | Swallows all `Exception` subclasses; callers cannot distinguish between I/O vs. authentication errors. | Catch specific exceptions (`IOException`, `StorageException`) and rethrow with context. |

### Future Enhancements  

1. **Streaming upload/download** – Use GCS’s streaming APIs to handle arbitrarily large files.  
2. **Metadata handling** – Store content type and custom metadata in `BlobInfo`.  
3. **Batch operations** – Implement `removeFiles` to delete all objects with a given prefix.  
4. **Folder abstraction** – Fully support hierarchical folders (even though GCS is flat) by consistently using prefixes.  
5. **Unit tests** – Mock the GCS client (`Storage`) to test without hitting the real cloud.  
6. **Configuration** – Externalize bucket name and project ID via Spring properties or a `CMSManager` implementation.  
7. **Documentation** – Add Javadoc comments, clarify the contract of `folderPath` and the helper methods.

---

### Overall Assessment  

The implementation provides a functional baseline for GCS‑backed content storage. It correctly uses the GCS SDK and follows a clean interface‑based design, making it easy to swap storage providers. However, several critical details (helper methods, folder support, streaming uploads, proper deletion) are missing or incomplete. Addressing the edge‑cases and enhancing error handling would make the component robust enough for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.content.gcp;

import java.io.IOException;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

import com.google.api.gax.paging.Page;
import com.google.cloud.storage.Blob;
import com.google.cloud.storage.BlobId;
import com.google.cloud.storage.BlobInfo;
import com.google.cloud.storage.Bucket;
import com.google.cloud.storage.Storage;
import com.google.cloud.storage.StorageOptions;
import com.google.cloud.storage.Blob.BlobSourceOption;
import com.google.cloud.storage.Storage.BlobField;
import com.google.cloud.storage.Storage.BucketGetOption;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.ContentAssetsManager;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * Content management for a given retailer using GC{ (Google Cloud Platform)
 * Cloud Storage with buckets
 * 
 * 
 * Linux/ Mac export GOOGLE_APPLICATION_CREDENTIALS="/path/to/file" For Windows
 * set GOOGLE_APPLICATION_CREDENTIALS="C:\path\to\file"
 * 
 * following this article: https://www.baeldung.com/java-google-cloud-storage
 * 
 * gsutil ls -L -b gs://shopizer-content
 * 
 * 
 * @author carlsamson
 *
 */
@Component("gcpContentAssetsManager")
public class GCPStaticContentAssetsManagerImpl implements ContentAssetsManager {

  private static final long serialVersionUID = 1L;

  private static final Logger LOGGER = LoggerFactory.getLogger(GCPStaticContentAssetsManagerImpl.class);

  @Autowired
  @Qualifier("gcpAssetsManager")
  private CMSManager cmsManager;

  @Override
	public OutputContentFile getFile(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType, String contentName)
			throws ServiceException {
    try {
      String bucketName = bucketName();
      // Instantiates a client
      Storage storage = StorageOptions.getDefaultInstance().getService();

      Blob blob = storage.get(BlobId.of(bucketName, nodePath(merchantStoreCode, fileContentType) + contentName));
      LOGGER.info("Content getFile");

      return getOutputContentFile(blob.getContent(BlobSourceOption.generationMatch()));

    } catch (Exception e) {
      LOGGER.error("Error while getting file", e);
      throw new ServiceException(e);
    }
  }

	@Override
	public List<String> getFileNames(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType)
			throws ServiceException {
		try {
			String bucketName = bucketName();
			Storage storage = StorageOptions.getDefaultInstance().getService();
			long bucketMetaGeneration = 42;
			Bucket bucket = storage.get(bucketName, BucketGetOption.metagenerationMatch(bucketMetaGeneration));
			Page<Blob> blobs = bucket.list(Storage.BlobListOption.prefix(nodePath(merchantStoreCode, fileContentType)),
				Storage.BlobListOption.fields(BlobField.NAME));
	
			List<String> fileNames = new ArrayList<String>();
	
			for (Blob blob : blobs.iterateAll()) {
				if (isInsideSubFolder(blob.getName()))
				continue;
				String mimetype = URLConnection.guessContentTypeFromName(blob.getName());
				if (!StringUtils.isBlank(mimetype)) {
				fileNames.add(getName(blob.getName()));
				}
	
			}
	
			LOGGER.info("Content get file names");
			return fileNames;
		} catch (Exception e) {
			LOGGER.error("Error while getting file names", e);
			throw new ServiceException(e);
		}
	}

	@Override
	public List<OutputContentFile> getFiles(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType)
			throws ServiceException {
		try {

			List<String> fileNames = getFileNames(merchantStoreCode, folderPath, fileContentType);
			List<OutputContentFile> files = new ArrayList<OutputContentFile>();
		
			for (String fileName : fileNames) {
				files.add(getFile(merchantStoreCode, folderPath, fileContentType, fileName));
			}
		
			LOGGER.info("Content get file names");
			return files;
		} catch (Exception e) {
		LOGGER.error("Error while getting file names", e);
		throw new ServiceException(e);
		}
	}

	@Override
	public void addFile(String merchantStoreCode, Optional<String> folderPath, InputContentFile inputStaticContentData) throws ServiceException {

		try {
			LOGGER.debug("Adding file " + inputStaticContentData.getFileName());
			String bucketName = bucketName();
	  
			String nodePath = nodePath(merchantStoreCode, inputStaticContentData.getFileContentType());
	  
			Storage storage = StorageOptions.getDefaultInstance().getService();
	  
			BlobId blobId = BlobId.of(bucketName, nodePath + inputStaticContentData.getFileName());
			BlobInfo blobInfo = BlobInfo.newBuilder(blobId).setContentType(inputStaticContentData.getFileContentType().name())
				.build();
			byte[] targetArray = new byte[inputStaticContentData.getFile().available()];
			inputStaticContentData.getFile().read(targetArray);
			storage.create(blobInfo, targetArray);
			LOGGER.info("Content add file");
		} catch (IOException e) {
			LOGGER.error("Error while adding file", e);
			throw new ServiceException(e);
		}
	}

	@Override
	public void addFiles(String merchantStoreCode, Optional<String> folderPath, List<InputContentFile> inputStaticContentDataList)
			throws ServiceException {
		if (CollectionUtils.isNotEmpty(inputStaticContentDataList)) {
			for (InputContentFile inputFile : inputStaticContentDataList) {
				this.addFile(merchantStoreCode, folderPath, inputFile);
			}
		}
	}

	@Override
	public void removeFile(String merchantStoreCode, FileContentType staticContentType, String fileName, Optional<String> folderPath)
			throws ServiceException {
		try {
			String bucketName = bucketName();
			Storage storage = StorageOptions.getDefaultInstance().getService();
			storage.delete(bucketName, nodePath(merchantStoreCode, staticContentType) + fileName);
		
			LOGGER.info("Remove file");
		} catch (final Exception e) {
			LOGGER.error("Error while removing file", e);
			throw new ServiceException(e);
		}			  
	}

	@Override
	public void removeFiles(String merchantStoreCode, Optional<String> folderPath) throws ServiceException {
		try {
			// get buckets
			String bucketName = bucketName();
	
			Storage storage = StorageOptions.getDefaultInstance().getService();
			storage.delete(bucketName, nodePath(merchantStoreCode));
	
			LOGGER.info("Remove folder");
		} catch (final Exception e) {
			LOGGER.error("Error while removing folder", e);
			throw new ServiceException(e);	
		}
	}

  
	public CMSManager getCmsManager() {
		return cmsManager;
	}

	public void setCmsManager(CMSManager cmsManager) {
		this.cmsManager = cmsManager;
	}


	@Override
	public void addFolder(String merchantStoreCode, String folderName, Optional<String> folderPath) throws ServiceException {
		// TODO Auto-generated method stub

	}

	@Override
	public void removeFolder(String merchantStoreCode, String folderName, Optional<String> folderPath) throws ServiceException {
		// TODO Auto-generated method stub

	}

	@Override
	public List<String> listFolders(String merchantStoreCode, Optional<String> folderPath) throws ServiceException {
		// TODO Auto-generated method stub
		return null;
	}

}



```
