# S3StaticContentAssetsManagerImpl.java

## Review

## 1. Summary  

**Purpose** – The class implements a static content manager that stores, retrieves and deletes files in an Amazon S3 bucket. It is an implementation of the `ContentAssetsManager` interface used by the CMS layer of the application.

**Key Components**  
| Component | Role |
|-----------|------|
| `S3StaticContentAssetsManagerImpl` | Singleton façade that delegates all S3 operations to the AWS SDK |
| `AmazonS3` client | Low‑level AWS SDK client used for CRUD operations |
| `InputContentFile / OutputContentFile` | DTOs that wrap the file stream, MIME type, etc. |
| `CMSManager` | Provides configuration such as the AWS region (via `getLocation()`) |

**Design patterns / libraries**  
* **Singleton** – `getInstance()` provides a single shared instance.  
* **Factory / Builder** – `AmazonS3ClientBuilder` is used to create the SDK client.  
* **Apache Commons** – `IOUtils`, `StringUtils`, `CollectionUtils`.  
* **Logging** – SLF4J with `LoggerFactory`.  

---

## 2. Detailed Description  

### Initialization  
1. The static field `fileManager` is lazily instantiated in `getInstance()` (not thread‑safe).  
2. No explicit AWS credentials are supplied; the SDK falls back to the default credential provider chain (environment variables, EC2 role, etc.).  
3. The bucket name is retrieved via a missing `bucketName()` method (likely inherited or omitted).  
4. The region is resolved through `getCmsManager().getLocation()`, defaulting to `DEFAULT_REGION_NAME` if empty.

### Runtime Behaviour  
For every public operation (`getFile`, `getFileNames`, `addFile`, etc.):

1. The bucket name and a new `AmazonS3` client are created.  
2. A S3 request (e.g., `getObject`, `putObject`, `listObjectsV2`) is executed.  
3. File content is converted to/from `byte[]` using `IOUtils`.  
4. Exceptions are caught generically, logged, and wrapped in a `ServiceException`.  

### Cleanup  
There is no explicit cleanup – the S3 client is created anew on each call and relies on the SDK’s internal pooling. Streams are not explicitly closed; `ByteArrayOutputStream` is not a resource that needs closing but the input streams from S3 are handled by `IOUtils` internally.

### Assumptions / Constraints  
* The S3 bucket is expected to exist (or will be created automatically).  
* File names are stored under a path prefix derived from `merchantStoreCode` and `FileContentType`.  
* Optional `folderPath` parameters are currently unused; all files are treated as being in the root of the type‑specific node.  
* The application runs in an environment where AWS credentials and network connectivity to S3 are available.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects | Notes |
|--------|---------|--------|---------|--------------|-------|
| `getInstance()` | Singleton factory | – | `S3StaticContentAssetsManagerImpl` | Initializes static instance | **Not thread‑safe** |
| `getFile(...)` | Retrieve a single file | `merchantStoreCode`, `folderPath` (unused), `fileContentType`, `contentName` | `OutputContentFile` | Reads file from S3 | Uses `IOUtils.toByteArray` (loads entire file into memory) |
| `getFileNames(...)` | List file names under a node | `merchantStoreCode`, `folderPath` (unused), `fileContentType` | `List<String>` | Queries S3 list API | Relies on helper methods `isInsideSubFolder`, `getName` (not shown) |
| `getFiles(...)` | Retrieve all files under a node | Same as above | `List<OutputContentFile>` | Loads each file into memory | Uses `ByteArrayOutputStream` per file |
| `addFile(...)` | Upload a single file | `merchantStoreCode`, `folderPath` (unused), `InputContentFile` | – | Creates S3 object with public read ACL | Uses `ObjectMetadata` for MIME type |
| `addFiles(...)` | Batch upload | `merchantStoreCode`, `folderPath` (unused), `List<InputContentFile>` | – | Calls `addFile` per entry | No transaction semantics |
| `removeFile(...)` | Delete a single object | `merchantStoreCode`, `fileContentType`, `fileName`, `folderPath` (unused) | – | Calls `deleteObject` | |
| `removeFiles(...)` | Delete a “folder” | `merchantStoreCode`, `folderPath` (unused) | – | Calls `deleteObject` on node path (does *not* delete all keys) | **Bug** – S3 has no real folders |
| `getBucket(...)` | Return or create bucket | `bucket_name` | `Bucket` | May call `createBucket` | |
| `createBucket(...)` | Create bucket if missing | `bucket_name` | `Bucket` | Calls `createBucket` or logs error | |
| `s3Client()` | Build AWS SDK client | – | `AmazonS3` | Builds a new client each call | **Potential inefficiency** |
| `regionName()` | Resolve region | – | `String` | Reads from `cmsManager` | |
| `addFolder(...)` / `removeFolder(...)` / `listFolders(...)` | Unimplemented | – | – | Stubbed out |
| `getCmsManager()` / `setCmsManager()` | Dependency injection | – | – | Allows configuration of CMSManager |

---

## 4. Dependencies  

| Library / SDK | Type | Notes |
|---------------|------|-------|
| **Amazon Web Services SDK (S3)** | Third‑party | Provides `AmazonS3ClientBuilder`, `S3Object`, etc. |
| **Apache Commons IO** | Third‑party | `IOUtils` for stream conversion. |
| **Apache Commons Lang** | Third‑party | `StringUtils` for string checks. |
| **Apache Commons Collections** | Third‑party | `CollectionUtils` for list checks. |
| **SLF4J** | Logging abstraction | `LoggerFactory` used for logging. |
| **SalesManager core modules** | Internal | `ContentAssetsManager`, `CMSManager`, DTOs. |

All dependencies are standard third‑party libraries; there are no platform‑specific assumptions beyond AWS credentials being available via the SDK’s provider chain.

---

## 5. Additional Notes  

### 5.1 Edge‑Cases & Bugs  
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| `removeFiles(...)` deletes only a single key (the node path) rather than all objects under that prefix. | Folder removal fails; orphan objects remain. | Iterate over `listObjectsV2` with the prefix and delete each key. |
| `folderPath` optional parameters are never used. | API surface is misleading; cannot manage sub‑folders. | Use the path to build the S3 key or drop the parameter. |
| Singleton `fileManager` is not thread‑safe. | Concurrency issues in a multi‑threaded environment. | Use `volatile` + double‑checked locking, or rely on a dependency injection framework. |
| `s3Client()` creates a new client for every operation. | Extra overhead; missing client pooling. | Create a single client in the constructor or via lazy init and cache it. |
| No explicit closing of input streams from S3 (`o.getObjectContent()`). | Minor resource leak (the SDK handles this, but explicit `finally` or try‑with‑resources is cleaner). | Wrap `o.getObjectContent()` in try‑with‑resources. |
| File operations load entire contents into memory (`byte[]`). | Out‑of‑memory for large files. | Stream directly to `OutputContentFile` if possible, or use transfer manager. |
| Missing `bucketName()` and `DEFAULT_REGION_NAME` definitions. | Compilation fails or misbehaves. | Provide proper implementation or configuration values. |
| Error handling is very generic (`catch (Exception e)`). | Loss of specific failure information. | Catch specific AWS exceptions, rethrow meaningful `ServiceException` with cause. |
| No Javadoc on public methods. | Reduced maintainability. | Add JavaDoc describing parameters, behavior, exceptions. |

### 5.2 Possible Enhancements  
1. **Dependency Injection** – Replace the manual singleton with a DI container (Spring, Guice) to inject `AmazonS3` and `CMSManager`.  
2. **Configuration Management** – Externalize bucket name and region via properties or environment variables.  
3. **Folder Support** – Implement `addFolder`, `removeFolder`, and `listFolders` correctly using S3 prefixes.  
4. **Transactional Upload** – Batch upload with rollback semantics if any upload fails.  
5. **Streaming API** – Expose an `InputStream` or `java.nio.file.Path` interface to avoid loading entire files into memory.  
6. **Metrics & Monitoring** – Log request latencies, success/failure counts.  
7. **Unit Tests** – Mock the `AmazonS3` client and test each operation.  
8. **Caching** – Cache bucket existence checks to avoid repeated `listBuckets` calls.  

### 5.3 Security Considerations  
* All uploaded files are given `CannedAccessControlList.PublicRead`.  Ensure that only trusted content is uploaded, or expose a configurable ACL.  
* If the application runs in an environment with IAM roles, make sure the role has only the necessary S3 permissions.  

---

### Bottom Line  
The class provides a straightforward mapping from CMS content operations to S3, but it has several shortcomings: missing folder handling, inefficient client creation, potential concurrency issues, and unimplemented methods. Addressing these concerns will improve robustness, performance, and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.content.aws;

import java.io.ByteArrayOutputStream;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.io.IOUtils;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.AmazonS3Exception;
import com.amazonaws.services.s3.model.Bucket;
import com.amazonaws.services.s3.model.CannedAccessControlList;
import com.amazonaws.services.s3.model.ListObjectsV2Request;
import com.amazonaws.services.s3.model.ListObjectsV2Result;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;
import com.amazonaws.services.s3.model.S3Object;
import com.amazonaws.services.s3.model.S3ObjectSummary;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.ContentAssetsManager;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * Static content management with S3
 * 
 * @author carlsamson
 *
 */
public class S3StaticContentAssetsManagerImpl implements ContentAssetsManager {

	private static final long serialVersionUID = 1L;

	private static final Logger LOGGER = LoggerFactory.getLogger(S3StaticContentAssetsManagerImpl.class);

	private static S3StaticContentAssetsManagerImpl fileManager = null;

	private CMSManager cmsManager;

	public static S3StaticContentAssetsManagerImpl getInstance() {

		if (fileManager == null) {
			fileManager = new S3StaticContentAssetsManagerImpl();
		}

		return fileManager;

	}

	@Override
	public OutputContentFile getFile(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType, String contentName)
			throws ServiceException {
		try {
			// get buckets
			String bucketName = bucketName();

			final AmazonS3 s3 = s3Client();

			S3Object o = s3.getObject(bucketName, nodePath(merchantStoreCode, fileContentType) + contentName);

			LOGGER.info("Content getFile");
			return getOutputContentFile(IOUtils.toByteArray(o.getObjectContent()));
		} catch (final Exception e) {
			LOGGER.error("Error while getting file", e);
			throw new ServiceException(e);

		}
	}

	@Override
	public List<String> getFileNames(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType)
			throws ServiceException {
		try {
			// get buckets
			String bucketName = bucketName();

			ListObjectsV2Request listObjectsRequest = new ListObjectsV2Request().withBucketName(bucketName)
					.withPrefix(nodePath(merchantStoreCode, fileContentType));

			List<String> fileNames = null;

			final AmazonS3 s3 = s3Client();
			ListObjectsV2Result results = s3.listObjectsV2(listObjectsRequest);
			List<S3ObjectSummary> objects = results.getObjectSummaries();
			for (S3ObjectSummary os : objects) {
				if (isInsideSubFolder(os.getKey())) {
					continue;
				}
				if (fileNames == null) {
					fileNames = new ArrayList<String>();
				}
				String mimetype = URLConnection.guessContentTypeFromName(os.getKey());
				if (!StringUtils.isBlank(mimetype)) {
					fileNames.add(getName(os.getKey()));
				}
			}

			LOGGER.info("Content get file names");
			return fileNames;
		} catch (final Exception e) {
			LOGGER.error("Error while getting file names", e);
			throw new ServiceException(e);

		}
	}

	@Override
	public List<OutputContentFile> getFiles(String merchantStoreCode, Optional<String> folderPath, FileContentType fileContentType)
			throws ServiceException {
		try {
			// get buckets
			String bucketName = bucketName();

			ListObjectsV2Request listObjectsRequest = new ListObjectsV2Request().withBucketName(bucketName)
					.withPrefix(nodePath(merchantStoreCode, fileContentType));

			List<OutputContentFile> files = null;
			final AmazonS3 s3 = s3Client();
			ListObjectsV2Result results = s3.listObjectsV2(listObjectsRequest);
			List<S3ObjectSummary> objects = results.getObjectSummaries();
			for (S3ObjectSummary os : objects) {
				if (files == null) {
					files = new ArrayList<OutputContentFile>();
				}
				String mimetype = URLConnection.guessContentTypeFromName(os.getKey());
				if (!StringUtils.isBlank(mimetype)) {
					S3Object o = s3.getObject(bucketName, os.getKey());
					byte[] byteArray = IOUtils.toByteArray(o.getObjectContent());
					ByteArrayOutputStream baos = new ByteArrayOutputStream(byteArray.length);
					baos.write(byteArray, 0, byteArray.length);
					OutputContentFile ct = new OutputContentFile();
					ct.setFile(baos);
					files.add(ct);
				}
			}

			LOGGER.info("Content getFiles");
			return files;
		} catch (final Exception e) {
			LOGGER.error("Error while getting files", e);
			throw new ServiceException(e);

		}
	}

	@Override
	public void addFile(String merchantStoreCode, Optional<String> folderPath, InputContentFile inputStaticContentData) throws ServiceException {

		try {
			// get buckets
			String bucketName = bucketName();

			String nodePath = nodePath(merchantStoreCode, inputStaticContentData.getFileContentType());

			final AmazonS3 s3 = s3Client();

			ObjectMetadata metadata = new ObjectMetadata();
			metadata.setContentType(inputStaticContentData.getMimeType());
			PutObjectRequest request = new PutObjectRequest(bucketName, nodePath + inputStaticContentData.getFileName(),
					inputStaticContentData.getFile(), metadata);
			request.setCannedAcl(CannedAccessControlList.PublicRead);

			s3.putObject(request);

			LOGGER.info("Content add file");
		} catch (final Exception e) {
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
			// get buckets
			String bucketName = bucketName();

			final AmazonS3 s3 = s3Client();
			s3.deleteObject(bucketName, nodePath(merchantStoreCode, staticContentType) + fileName);

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

			final AmazonS3 s3 = s3Client();
			s3.deleteObject(bucketName, nodePath(merchantStoreCode));

			LOGGER.info("Remove folder");
		} catch (final Exception e) {
			LOGGER.error("Error while removing folder", e);
			throw new ServiceException(e);

		}

	}

	private Bucket getBucket(String bucket_name) {
		final AmazonS3 s3 = s3Client();
		Bucket named_bucket = null;
		List<Bucket> buckets = s3.listBuckets();
		for (Bucket b : buckets) {
			if (b.getName().equals(bucket_name)) {
				named_bucket = b;
			}
		}

		if (named_bucket == null) {
			named_bucket = createBucket(bucket_name);
		}

		return named_bucket;
	}

	private Bucket createBucket(String bucket_name) {
		final AmazonS3 s3 = s3Client();
		Bucket b = null;
		if (s3.doesBucketExistV2(bucket_name)) {
			System.out.format("Bucket %s already exists.\n", bucket_name);
			b = getBucket(bucket_name);
		} else {
			try {
				b = s3.createBucket(bucket_name);
			} catch (AmazonS3Exception e) {
				System.err.println(e.getErrorMessage());
			}
		}
		return b;
	}

	/**
	 * Builds an amazon S3 client
	 * 
	 * @return
	 */
	private AmazonS3 s3Client() {
		String region = regionName();
		LOGGER.debug("AWS CMS Using region " + region);

		return AmazonS3ClientBuilder.standard().withRegion(region) // The
																			// first
																			// region
																			// to
																			// try
																			// your
																			// request
																			// against
				.build();
	}

	private String regionName() {
		String regionName = getCmsManager().getLocation();
		if (StringUtils.isBlank(regionName)) {
			regionName = DEFAULT_REGION_NAME;
		}
		return regionName;
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
	public List<String> listFolders(String merchantStoreCode, Optional<String> path) throws ServiceException {
		// TODO Auto-generated method stub
		return null;
	}

}



```
