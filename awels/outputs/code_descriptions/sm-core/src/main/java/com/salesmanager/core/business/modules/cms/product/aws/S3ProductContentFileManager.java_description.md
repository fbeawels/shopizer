# S3ProductContentFileManager.java

## Review

## 1. Summary  
`S3ProductContentFileManager` is a concrete implementation of the `ProductAssetsManager` interface that stores, retrieves and deletes product‑related files (primarily images) in Amazon S3.  
Key responsibilities:

| Responsibility | Implementation |
|----------------|----------------|
| **Singleton access** | `getInstance()` returns a single shared instance. |
| **Bucket configuration** | Reads bucket name & region from a `CMSManager`; falls back to defaults if missing. |
| **Path construction** | Builds S3 keys that mirror the product hierarchy (`products/{store}/{sku}/`). |
| **CRUD operations** | `addProductImage`, `removeProductImage`, `removeProductImages`, `removeImages` and `getImages`. |
| **Utility helpers** | `nodePath(...)`, `getName`, `indexOfLastSeparator`. |

The class relies on the AWS SDK (`com.amazonaws.services.s3`) and several application‑specific classes (`CMSManager`, `ProductImage`, `ImageContentFile`, etc.).  

Design-wise, it uses the **Singleton** pattern (not thread‑safe) and the **Facade** style for S3 interactions, abstracting away AWS details behind a business‑level API.

---

## 2. Detailed Description  

### 2.1. Initialization
- A static field `fileManager` holds the singleton instance; the first call to `getInstance()` creates it.
- `cmsManager` must be injected via `setCmsManager` before any S3 operation; otherwise bucket/region resolution will fail (potential NPE).

### 2.2. Runtime Flow  
1. **Bucket & Region**  
   - `bucketName()` → `cmsManager.getRootName()` (fallback to `"shopizer-content"`).  
   - `regionName()` → `cmsManager.getLocation()` (fallback to `"us-east-1"`).  
   - `s3Client()` constructs a new `AmazonS3` client on every call (no reuse).

2. **Path Building**  
   - `nodePath(store)` → `"products/<store>/"`.  
   - `nodePath(store, sku)` → `"products/<store>/<sku>/"`.  
   - `nodePath(store, sku, ImageContentFile)` appends `"SMALL"` or `"LARGE"` depending on the content type.

3. **CRUD Operations**  
   - **Add**: Builds a `PutObjectRequest` with public read ACL and writes the `ImageContentFile` bytes.  
   - **Remove**: Calls `deleteObject` on a key that usually represents a folder prefix; this deletes **only one object**, not all images under that prefix.  
   - **Retrieve**: Lists all objects under a prefix, streams each into a `ByteArrayOutputStream`, wraps it in an `OutputContentFile` and returns a list.

4. **Error Handling**  
   - Every method catches `Exception`, logs, and rethrows as a `ServiceException`.  
   - Some log messages are inaccurate (e.g., “Error while removing file” when deleting a folder).

### 2.3. Assumptions & Constraints  
- The S3 bucket already exists; otherwise `s3Client().createBucket` is *not* invoked because `getBucket`/`createBucket` are unused.  
- Pagination is not handled; `listObjectsV2` will only return up to 1000 keys.  
- The caller is responsible for injecting a valid `CMSManager`.  
- No thread‑safety guarantees on the singleton.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getInstance()` | Retrieve singleton instance. | – | `S3ProductContentFileManager` | Creates instance if null. |
| `getImages(String, FileContentType)` | Return all images for a store. | `merchantStoreCode`, `imageContentType` | `List<OutputContentFile>` | Reads from S3. |
| `removeImages(String)` | Delete images for a store. | `merchantStoreCode` | – | Calls `deleteObject` on folder key. |
| `removeProductImage(ProductImage)` | Delete a specific product image. | `productImage` | – | `deleteObject` on key. |
| `removeProductImages(Product)` | Delete all images for a product. | `product` | – | `deleteObject` on product key. |
| `getProductImage(...)` (4 overloads) | Unimplemented stubs. | – | `null` | – |
| `addProductImage(ProductImage, ImageContentFile)` | Store a new image. | `productImage`, `contentImage` | – | `putObject` to S3. |
| `getBucket(String)` | Return or create a bucket. | `bucket_name` | `Bucket` | May call `createBucket`. |
| `createBucket(String)` | Create bucket if not exists. | `bucket_name` | `Bucket` | Uses S3 client. |
| `s3Client()` | Build new S3 client. | – | `AmazonS3` | – |
| `bucketName()` | Resolve bucket name. | – | `String` | – |
| `regionName()` | Resolve region. | – | `String` | – |
| `nodePath(String)`, `nodePath(String,String)`, `nodePath(String,String,ImageContentFile)` | Build S3 key prefixes. | store, sku, content | `String` | – |
| `getName(String)` | Get file name from path. | `filename` | `String` | – |
| `indexOfLastSeparator(String)` | Find last path separator. | `filename` | `int` | – |
| `getCmsManager()` / `setCmsManager(CMSManager)` | Accessor for CMS manager. | – | – | – |

---

## 4. Dependencies  

| Library / Package | Type | Purpose |
|-------------------|------|---------|
| `com.amazonaws.services.s3.*` | Third‑party | Interact with Amazon S3 (client, models). |
| `org.apache.commons.io.IOUtils` | Third‑party | Stream utilities for reading S3 objects. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | String handling utilities. |
| `org.slf4j.*` | Third‑party | Logging. |
| `com.salesmanager.core.*` | Application | Domain models (`Product`, `ProductImage`, etc.) and custom exceptions. |

All dependencies are either standard (Java SE) or well‑known third‑party libraries.

---

## 5. Additional Notes & Recommendations  

### 5.1. Thread‑Safety & Singleton
- The `fileManager` field is accessed without synchronization. In a multi‑threaded environment, multiple instances may be created.  
- **Fix**: Use double‑checked locking, an `enum` singleton, or dependency injection frameworks.

### 5.2. Bucket Management
- `bucketName()` relies on `cmsManager` but never creates the bucket. The unused `getBucket`/`createBucket` methods suggest this was considered but omitted.  
- **Fix**: Either call `createBucket` during initialization or document that the bucket must pre‑exist.

### 5.3. Deletion Logic
- `deleteObject` only deletes a single key. If the intention is to delete all objects under a prefix, use `listObjectsV2` + `deleteObjects` with a batch of `DeleteObject` requests.  
- **Fix**: Implement recursive delete or batch delete.

### 5.4. Pagination
- `listObjectsV2` returns at most 1000 keys. For larger product catalogs, subsequent pages must be fetched via `ContinuationToken`.  
- **Fix**: Loop until `isTruncated()` is false.

### 5.5. Memory Usage
- Reading entire files into a `byte[]` then into a `ByteArrayOutputStream` duplicates data in memory.  
- **Fix**: Pass the `InputStream` directly or stream the response to the caller.

### 5.6. Logging Accuracy
- Several catch blocks log “Error while removing file” even when a folder is being deleted.  
- **Fix**: Update log messages to reflect the operation.

### 5.7. Unimplemented Methods
- The `getProductImage` overloads return `null`.  
- **Fix**: Provide concrete implementations or throw `UnsupportedOperationException`.

### 5.8. Path Separator Constants
- `UNIX_SEPARATOR`/`WINDOWS_SEPARATOR` are defined but never used.  
- Consider normalizing paths with `java.nio.file.Paths` or using `Constants.SLASH`.

### 5.9. ACL & Metadata
- Public read ACL is hard‑coded; this may not be desired in all deployments.  
- **Fix**: Make ACL configurable.

### 5.10. Error Handling
- Swallowing generic `Exception` hides specific AWS exceptions (e.g., `AmazonS3Exception`).  
- **Fix**: Catch specific exceptions and provide richer context.

---

**Overall Verdict**  
The class provides a reasonable abstraction over S3 for product assets but contains several practical issues (thread‑safety, bucket creation, deletion semantics, pagination, and incomplete methods). Addressing the above points will make the component robust, maintainable, and suitable for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.product.aws;

import java.io.ByteArrayOutputStream;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.List;
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
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.business.modules.cms.product.ProductAssetsManager;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * Product content file manager with AWS S3
 * 
 * @author carlsamson
 *
 */
public class S3ProductContentFileManager
    implements ProductAssetsManager {

  /**
   * 
   */
  private static final long serialVersionUID = 1L;



  private static final Logger LOGGER = LoggerFactory.getLogger(S3ProductContentFileManager.class);



  private static S3ProductContentFileManager fileManager = null;

  private static String DEFAULT_BUCKET_NAME = "shopizer-content";
  private static String DEFAULT_REGION_NAME = "us-east-1";
  private static final String ROOT_NAME = "products";

  private static final char UNIX_SEPARATOR = '/';
  private static final char WINDOWS_SEPARATOR = '\\';


  private final static String SMALL = "SMALL";
  private final static String LARGE = "LARGE";

  private CMSManager cmsManager;

  public static S3ProductContentFileManager getInstance() {

    if (fileManager == null) {
      fileManager = new S3ProductContentFileManager();
    }

    return fileManager;

  }

  @Override
  public List<OutputContentFile> getImages(String merchantStoreCode,
      FileContentType imageContentType) throws ServiceException {
    try {
      // get buckets
      String bucketName = bucketName();



      ListObjectsV2Request listObjectsRequest = new ListObjectsV2Request()
          .withBucketName(bucketName).withPrefix(nodePath(merchantStoreCode));

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

      return files;
    } catch (final Exception e) {
      LOGGER.error("Error while getting files", e);
      throw new ServiceException(e);

    }
  }

  @Override
  public void removeImages(String merchantStoreCode) throws ServiceException {
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

  @Override
  public void removeProductImage(ProductImage productImage) throws ServiceException {
    try {
      // get buckets
      String bucketName = bucketName();

      final AmazonS3 s3 = s3Client();
      s3.deleteObject(bucketName, nodePath(productImage.getProduct().getMerchantStore().getCode(),
          productImage.getProduct().getSku()) + productImage.getProductImage());

      LOGGER.info("Remove file");
    } catch (final Exception e) {
      LOGGER.error("Error while removing file", e);
      throw new ServiceException(e);

    }

  }

  @Override
  public void removeProductImages(Product product) throws ServiceException {
    try {
      // get buckets
      String bucketName = bucketName();

      final AmazonS3 s3 = s3Client();
      s3.deleteObject(bucketName, nodePath(product.getMerchantStore().getCode(), product.getSku()));

      LOGGER.info("Remove file");
    } catch (final Exception e) {
      LOGGER.error("Error while removing file", e);
      throw new ServiceException(e);

    }

  }

  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName) throws ServiceException {
    // TODO Auto-generated method stub
    return null;
  }

  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName, ProductImageSize size) throws ServiceException {
    // TODO Auto-generated method stub
    return null;
  }

  @Override
  public OutputContentFile getProductImage(ProductImage productImage) throws ServiceException {
    // TODO Auto-generated method stub
    return null;
  }

  @Override
  public List<OutputContentFile> getImages(Product product) throws ServiceException {
    return null;
  }

  @Override
  public void addProductImage(ProductImage productImage, ImageContentFile contentImage)
      throws ServiceException {


    try {
      // get buckets
      String bucketName = bucketName();
      final AmazonS3 s3 = s3Client();

      String nodePath = this.nodePath(productImage.getProduct().getMerchantStore().getCode(),
          productImage.getProduct().getSku(), contentImage);


      ObjectMetadata metadata = new ObjectMetadata();
      metadata.setContentType(contentImage.getMimeType());

      PutObjectRequest request = new PutObjectRequest(bucketName,
          nodePath + productImage.getProductImage(), contentImage.getFile(), metadata);
      request.setCannedAcl(CannedAccessControlList.PublicRead);


      s3.putObject(request);


      LOGGER.info("Product add file");

    } catch (final Exception e) {
      LOGGER.error("Error while removing file", e);
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

    return AmazonS3ClientBuilder.standard().withRegion(regionName()) // The first region to
                                                                            // try your request
                                                                            // against
        .build();
  }

  private String bucketName() {
    String bucketName = getCmsManager().getRootName();
    if (StringUtils.isBlank(bucketName)) {
      bucketName = DEFAULT_BUCKET_NAME;
    }
    return bucketName;
  }

  private String regionName() {
    String regionName = getCmsManager().getLocation();
    if (StringUtils.isBlank(regionName)) {
      regionName = DEFAULT_REGION_NAME;
    }
    return regionName;
  }

  private String nodePath(String store) {
    return new StringBuilder().append(ROOT_NAME).append(Constants.SLASH).append(store)
        .append(Constants.SLASH).toString();
  }

  private String nodePath(String store, String product) {

    StringBuilder sb = new StringBuilder();
    // node path
    String nodePath = nodePath(store);
    sb.append(nodePath);

    // product path
    sb.append(product).append(Constants.SLASH);
    return sb.toString();

  }

  private String nodePath(String store, String product, ImageContentFile contentImage) {

    StringBuilder sb = new StringBuilder();
    // node path
    String nodePath = nodePath(store, product);
    sb.append(nodePath);

    // small large
    if (contentImage.getFileContentType().name().equals(FileContentType.PRODUCT.name())) {
      sb.append(SMALL);
    } else if (contentImage.getFileContentType().name().equals(FileContentType.PRODUCTLG.name())) {
      sb.append(LARGE);
    }

    return sb.append(Constants.SLASH).toString();


  }

  public static String getName(String filename) {
    if (filename == null) {
      return null;
    }
    int index = indexOfLastSeparator(filename);
    return filename.substring(index + 1);
  }

  public static int indexOfLastSeparator(String filename) {
    if (filename == null) {
      return -1;
    }
    int lastUnixPos = filename.lastIndexOf(UNIX_SEPARATOR);
    int lastWindowsPos = filename.lastIndexOf(WINDOWS_SEPARATOR);
    return Math.max(lastUnixPos, lastWindowsPos);
  }



  public CMSManager getCmsManager() {
    return cmsManager;
  }

  public void setCmsManager(CMSManager cmsManager) {
    this.cmsManager = cmsManager;
  }


}



```
