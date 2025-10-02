# GCPProductContentFileManager.java

## Review

## 1. Summary  

`GCPProductContentFileManager` is a Spring component that implements the `ProductAssetsManager` interface.  
Its purpose is to store, retrieve and delete product images (and related media) in **Google Cloud Storage (GCS)**.  
The manager uses a root bucket name supplied by an injected `CMSManager`; if none is configured it falls back to the hard‑coded `"shopizer"` bucket.  

Key responsibilities  

| Feature | Description |
|---------|-------------|
| **Get image** | Retrieve a product image (full size or a specific `ProductImageSize`). |
| **List images** | List all images for a merchant store or a product. |
| **Add image** | Upload a new product image (different sizes are derived from the image type). |
| **Delete image(s)** | Remove single or multiple images (by product or by merchant). |

The class relies on GCS client libraries (`com.google.cloud.storage.*`) and Apache Commons IO / Lang utilities.  
No explicit design pattern is used beyond the typical *Repository*‑like data access logic.

---

## 2. Detailed Description  

### Flow of Execution  

1. **Initialization**  
   * Spring injects a `CMSManager` (`gcpAssetsManager`).  
   * `DEFAULT_BUCKET_NAME` is used when `CMSManager` returns an empty value.

2. **Runtime**  
   * Every public method obtains a `Storage` instance via `StorageOptions.getDefaultInstance().getService()`.  
   * For write operations it checks/creates the bucket (`bucketExists / createBucket`).  
   * File paths are constructed via two helper overloads:
     * `filePath(merchant, sku, contentImage)` – used for *upload* and *list* operations.
     * `filePath(merchant, sku, size, fileName)` – used for *download* and *delete* operations.  

3. **Cleanup**  
   * The code manually closes `InputStream` objects in a `finally` block; the streams opened by GCS (`Channels.newInputStream(reader)`) are closed in the same block.  
   * No explicit close is performed for `OutputStream` objects – they are garbage‑collected but should be closed to free resources promptly.  

### Key Assumptions / Constraints  

| Aspect | Assumption | Impact |
|--------|------------|--------|
| Bucket name | Must exist or be creatable. | Failing to create a bucket causes the whole operation to fail silently (e.g., add image). |
| Image size mapping | `FileContentType.PRODUCT` maps to `"SMALL"` and `FileContentType.PRODUCTLG` maps to `"LARGE"`. | Any other content types will produce a path ending with a trailing slash only, leading to missing images. |
| File type | Hard‑coded `"image/jpeg"` when uploading. | Other image formats (PNG, GIF, SVG, etc.) will be stored with a wrong MIME type. |
| Memory usage | Entire image is read into memory (`ByteArrayOutputStream`). | Large images can exhaust heap. |
| Serialization | `serialVersionUID` is present though the class does **not** implement `Serializable`. | Unnecessary field that may mislead maintainers. |

### Architecture & Design Choices  

* **Single responsibility** – the class focuses on GCS operations only.  
* **Dependency injection** – `CMSManager` is injected, enabling testability.  
* **Minimal external dependencies** – uses GCS SDK and Apache Commons, both stable and widely used.  
* **Error handling** – wrapped in a custom `ServiceException`.  
* **Hard‑coded constants** – `DEFAULT_BUCKET_NAME`, `SMALL`, `LARGE` are static final strings; they could be externalised.  

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `OutputContentFile getProductImage(String, String, String)` | **TODO** – should retrieve an image by store, product and filename. | `merchantStoreCode`, `productCode`, `imageName` | `OutputContentFile` or `null` | None |
| `OutputContentFile getProductImage(String, String, String, ProductImageSize)` | Retrieves a product image of a specific size. | `merchantStoreCode`, `productCode`, `imageName`, `size` | `OutputContentFile` or `null` | Reads from GCS |
| `OutputContentFile getProductImage(ProductImage)` | **TODO** – should retrieve an image from a `ProductImage` instance. | `productImage` | `OutputContentFile` | None |
| `List<OutputContentFile> getImages(Product)` | **TODO** – should list all images for a product. | `product` | List or `null` | None |
| `List<OutputContentFile> getImages(String, FileContentType)` | Lists all images for a merchant store (by prefix). | `merchantStoreCode`, `imageContentType` | List of `OutputContentFile` | Reads all blobs matching the prefix |
| `void addProductImage(ProductImage, ImageContentFile)` | Uploads an image to GCS. | `productImage`, `contentImage` | None | Creates bucket if necessary, writes file, sets ACL |
| `void removeProductImage(ProductImage)` | Deletes all sizes for a product image. | `productImage` | None | Calls `storage.delete` twice (SMALL/LARGE) |
| `void removeProductImages(Product)` | Deletes all blobs with prefix equal to product SKU. | `product` | None | Deletes each blob found |
| `void removeImages(String)` | Deletes all blobs with prefix equal to merchant store code. | `merchantStoreCode` | None | Deletes each blob found |
| `String bucketName()` | Resolves the bucket name using `CMSManager` or default. | None | Bucket name | None |
| `boolean bucketExists(Storage, String)` | Checks existence of a bucket. | `storage`, `bucketName` | `true/false` | None |
| `Bucket createBucket(Storage, String)` | Creates a new bucket. | `storage`, `bucketName` | `Bucket` | None |
| `String filePath(String, String, FileContentType)` | Builds a storage path for an image size. | `merchant`, `sku`, `contentImage` | Path string | None |
| `String filePath(String, String, String, String)` | Builds a storage path for a specific file. | `merchant`, `sku`, `size`, `fileName` | Path string | None |

**Reusable / Utility Methods**  
* `bucketName()`, `bucketExists()`, `createBucket()` – common GCS bucket handling.  
* `filePath()` overloads – centralises the path building logic, reducing duplication.

---

## 4. Dependencies  

| Dependency | Type | Purpose |
|------------|------|---------|
| `com.google.cloud:google-cloud-storage` | Third‑party | GCS client library |
| `org.apache.commons:commons-io` | Third‑party | `IOUtils` for stream copy / byte array conversion |
| `org.apache.commons:commons-lang3` | Third‑party | `StringUtils` for blank checks |
| `org.slf4j:slf4j-api` | Third‑party | Logging |
| Spring (`@Component`, `@Autowired`) | Framework | Dependency injection, component scanning |
| Custom | `CMSManager`, `ProductAssetsManager`, model classes (`Product`, `ProductImage`, etc.) | Application‑specific business logic and data models |

All dependencies are well‑maintained and platform‑agnostic. No native OS or JDK version assumptions are visible.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation of concerns** – GCS logic isolated in one class.  
* **Testability** – `CMSManager` injection and use of interfaces make unit testing possible.  
* **Extensible** – Adding new image sizes or storage providers would involve adding new path variants or a different implementation of `ProductAssetsManager`.  

### Potential Issues & Edge Cases  

1. **Unimplemented methods** – Several `TODO` stubs (`getProductImage(...)`, `getProductImage(ProductImage)`, `getImages(Product)`) return `null`. This will cause `NullPointerException` at runtime if called.  
2. **Missing null checks** –  
   * `Blob blob = storage.get(...)` may return `null`; subsequent calls (`blob.reader()`) will throw NPE.  
   * `blob.getName()` used but may be `null`.  
3. **Resource leaks** –  
   * `InputStream` from `Channels.newInputStream(reader)` is closed in `finally`, but `ByteArrayOutputStream` is never closed.  
   * When looping over blobs in `getImages(...)`, `inputStream` is overwritten without closing the previous stream.  
4. **Memory consumption** – Entire file is read into a `ByteArrayOutputStream`. For large images, this can lead to `OutOfMemoryError`. Streaming directly to the destination (e.g., `OutputContentFile` could expose an `OutputStream`) would be safer.  
5. **Hard‑coded MIME type** – `BlobInfo` uses `"image/jpeg"` regardless of the actual file type. If PNG/GIF images are uploaded, the MIME type will be incorrect.  
6. **ACL creation in `addProductImage`** – Re‑creating the ACL for every upload may be redundant; GCS defaults to private unless a bucket policy is set.  
7. **`serialVersionUID` present on a non‑Serializable class** – Unnecessary and may mislead maintainers.  
8. **Inconsistent path logic** –  
   * `filePath(merchant, sku, contentImage)` ends with a trailing slash and does not append the filename, used only for listing/uploading.  
   * `filePath(merchant, sku, size, fileName)` contains the full filename. The mapping between these two may lead to bugs if a different content type is used.  
9. **Thread‑safety of `Storage`** – The GCS client is thread‑safe, but the class itself is not synchronized; if the component is used in a highly concurrent environment, the `FileContentFile` streams could be shared inadvertently.  

### Suggested Improvements  

| Area | Recommendation |
|------|----------------|
| **Implement missing methods** | Provide concrete logic or throw `UnsupportedOperationException` with clear message. |
| **Null handling** | Guard against `null` blobs; return `Optional<OutputContentFile>` or throw a domain exception. |
| **Resource management** | Use **try‑with‑resources** for all streams (`InputStream`, `ByteArrayOutputStream`). |
| **Streaming** | Consider exposing an `OutputStream` in `OutputContentFile` and writing directly to it. |
| **MIME type** | Infer MIME type from file extension or use `contentImage.getContentType()`. |
| **ACLs** | Set bucket policy once or rely on the bucket’s default ACL; avoid per‑object ACL recreation. |
| **Remove `serialVersionUID`** | Delete the field to avoid confusion. |
| **Configuration** | Externalise bucket name, default bucket, and size mappings via properties. |
| **Logging** | Add more contextual logs (e.g., bucket name, operation type). |
| **Error handling** | Distinguish between recoverable and fatal errors; provide detailed `ServiceException` messages. |
| **Unit tests** | Add tests for all public methods, mocking `Storage` and `CMSManager`. |

By addressing these points the class will become more robust, efficient, and easier to maintain.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.product.gcp;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.channels.Channels;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

import org.apache.commons.io.IOUtils;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import com.google.api.gax.paging.Page;
import com.google.cloud.ReadChannel;
import com.google.cloud.storage.Acl;
import com.google.cloud.storage.Acl.Role;
import com.google.cloud.storage.Acl.User;
import com.google.cloud.storage.Blob;
import com.google.cloud.storage.BlobId;
import com.google.cloud.storage.BlobInfo;
import com.google.cloud.storage.Bucket;
import com.google.cloud.storage.BucketInfo;
import com.google.cloud.storage.Storage;
import com.google.cloud.storage.Storage.BlobListOption;
import com.google.cloud.storage.Storage.BucketField;
import com.google.cloud.storage.Storage.BucketGetOption;
import com.google.cloud.storage.StorageOptions;
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

@Component("gcpProductAssetsManager")
public class GCPProductContentFileManager implements ProductAssetsManager {
  
  @Autowired 
  private CMSManager gcpAssetsManager;
  
  private static String DEFAULT_BUCKET_NAME = "shopizer";
  
  private static final Logger LOGGER = LoggerFactory.getLogger(GCPProductContentFileManager.class);

  

  private final static String SMALL = "SMALL";
  private final static String LARGE = "LARGE";

  /**
   * 
   */
  private static final long serialVersionUID = 1L;


  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName) throws ServiceException {
    // TODO Auto-generated method stub
    
 
    
    return null;
  }

  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName, ProductImageSize size) throws ServiceException {
    InputStream inputStream = null;
    try {
      Storage storage = StorageOptions.getDefaultInstance().getService();
      
      String bucketName = bucketName();
      
      if(!this.bucketExists(storage, bucketName)) {
        return null;
      }

      Blob blob = storage.get(BlobId.of(bucketName, filePath(merchantStoreCode,productCode, size.name(), imageName)));

      ReadChannel reader = blob.reader();
      
      inputStream = Channels.newInputStream(reader);
      ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
      IOUtils.copy(inputStream, outputStream);
      OutputContentFile ct = new OutputContentFile();
      ct.setFile(outputStream);
      ct.setFileName(blob.getName());

      

      return ct;
    } catch (final Exception e) {
      LOGGER.error("Error while getting files", e);
      throw new ServiceException(e);
  
    } finally {
      if(inputStream!=null) {
        try {
          inputStream.close();
        } catch(Exception ignore) {}
      }
      
    }
  
  }

  @Override
  public OutputContentFile getProductImage(ProductImage productImage) throws ServiceException {

    return null;
    
  }

  @Override
  public List<OutputContentFile> getImages(Product product) throws ServiceException {
    // TODO Auto-generated method stub
    return null;
  }

  /**
   * List files
   */
  @Override
  public List<OutputContentFile> getImages(String merchantStoreCode,
      FileContentType imageContentType) throws ServiceException {
    
    InputStream inputStream = null;
    try {
      Storage storage = StorageOptions.getDefaultInstance().getService();
      
      String bucketName = bucketName();
      
      if(!this.bucketExists(storage, bucketName)) {
        return null;
      }
      
      Page<Blob> blobs =
          storage.list(
              bucketName, BlobListOption.currentDirectory(), BlobListOption.prefix(merchantStoreCode));

      List<OutputContentFile> files = new ArrayList<OutputContentFile>();
      for (Blob blob : blobs.iterateAll()) {
        blob.getName();
        ReadChannel reader = blob.reader();
        inputStream = Channels.newInputStream(reader);
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        IOUtils.copy(inputStream, outputStream);
        OutputContentFile ct = new OutputContentFile();
        ct.setFile(outputStream);
        files.add(ct);
      }

      return files;
    } catch (final Exception e) {
      LOGGER.error("Error while getting files", e);
      throw new ServiceException(e);
  
    } finally {
      if(inputStream!=null) {
        try {
          inputStream.close();
        } catch(Exception ignore) {}
      }
      
    }
  }

  @Override
  public void addProductImage(ProductImage productImage, ImageContentFile contentImage)
      throws ServiceException {
    
    Storage storage = StorageOptions.getDefaultInstance().getService();
    
    String bucketName = bucketName();

    if(!this.bucketExists(storage, bucketName)) {
      createBucket(storage, bucketName);
    }
    
    //build filename
    StringBuilder fileName = new StringBuilder()
        .append(filePath(productImage.getProduct().getMerchantStore().getCode(), productImage.getProduct().getSku(), contentImage.getFileContentType()))
        .append(productImage.getProductImage());
    
    
      try {
        byte[] targetArray = IOUtils.toByteArray(contentImage.getFile());
        BlobId blobId = BlobId.of(bucketName, fileName.toString());
        BlobInfo blobInfo = BlobInfo.newBuilder(blobId).setContentType("image/jpeg").build();
        storage.create(blobInfo, targetArray);
        Acl acl = storage.createAcl(blobId, Acl.of(User.ofAllUsers(), Role.READER));
      } catch (IOException ioe) {
        throw new ServiceException(ioe);
      }

    
  }

  @Override
  public void removeProductImage(ProductImage productImage) throws ServiceException {
    
    //delete all image sizes
    Storage storage = StorageOptions.getDefaultInstance().getService();

    List<String> sizes = Arrays.asList(SMALL, LARGE);
    for(String size : sizes) {
      String filePath = filePath(productImage.getProduct().getMerchantStore().getCode(), productImage.getProduct().getSku(), size, productImage.getProductImage());
      BlobId blobId = BlobId.of(bucketName(), filePath);
      if(blobId==null) {
        LOGGER.info("Image path " + filePath + " does not exist");
        return;
        //throw new ServiceException("Image not found " + productImage.getProductImage());
      }
      boolean deleted = storage.delete(blobId);
      if (!deleted) {
        LOGGER.error("Cannot delete image [" + productImage.getProductImage() + "]");
      }
    }
  
  }

  @Override
  public void removeProductImages(Product product) throws ServiceException {

    
    Storage storage = StorageOptions.getDefaultInstance().getService();
    
    String bucketName = bucketName();

    Page<Blob> blobs =
        storage.list(
            bucketName, BlobListOption.currentDirectory(), BlobListOption.prefix(product.getSku()));

    
    for (Blob blob : blobs.iterateAll()) {
      // do something with the blob
      storage.delete(blob.getBlobId());
    }
    

  }

  @Override
  public void removeImages(String merchantStoreCode) throws ServiceException {
    Storage storage = StorageOptions.getDefaultInstance().getService();
    
    String bucketName = bucketName();

    Page<Blob> blobs =
        storage.list(
            bucketName, BlobListOption.currentDirectory(), BlobListOption.prefix(merchantStoreCode));

    
    for (Blob blob : blobs.iterateAll()) {
      // do something with the blob
      storage.delete(blob.getBlobId());
    }

  }
  
  private String bucketName() {
    String bucketName = gcpAssetsManager.getRootName();
    if (StringUtils.isBlank(bucketName)) {
      bucketName = DEFAULT_BUCKET_NAME;
    }
    return bucketName;
  }
  
  private boolean bucketExists(Storage storage, String bucketName) {
    Bucket bucket = storage.get(bucketName, BucketGetOption.fields(BucketField.NAME));
    if (bucket == null || !bucket.exists()) {
      return false;
    }
    return true;
  }
  
  private Bucket createBucket(Storage storage, String bucketName) {
    return storage.create(BucketInfo.of(bucketName));
  }
  
  private String filePath(String merchant, String sku, FileContentType contentImage) {
      StringBuilder sb = new StringBuilder();
      sb.append("products").append(Constants.SLASH);
      sb.append(merchant)
      .append(Constants.SLASH).append(sku).append(Constants.SLASH);

      // small large
      if (contentImage.name().equals(FileContentType.PRODUCT.name())) {
        sb.append(SMALL);
      } else if (contentImage.name().equals(FileContentType.PRODUCTLG.name())) {
        sb.append(LARGE);
      }

      return sb.append(Constants.SLASH).toString();
    
  }
  
  private String filePath(String merchant, String sku, String size, String fileName) {
    StringBuilder sb = new StringBuilder();
    sb.append("products").append(Constants.SLASH);
    sb.append(merchant)
    .append(Constants.SLASH).append(sku).append(Constants.SLASH);
    
    sb.append(size);
    sb.append(Constants.SLASH).append(fileName);

    return sb.toString();
  
  }


}


```
