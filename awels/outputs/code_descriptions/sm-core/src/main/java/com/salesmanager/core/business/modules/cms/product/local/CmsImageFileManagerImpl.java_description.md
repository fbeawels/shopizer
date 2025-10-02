# CmsImageFileManagerImpl.java

## Review

## 1. Summary  

**Purpose**  
`CmsImageFileManagerImpl` is a concrete implementation of the `ProductAssetsManager` interface that stores, removes, and (intended) retrieves product image files on a local file system.  The images are organised under a root directory that is defined by a `CMSManager`/`LocalCacheManagerImpl`.  

**Key components**  

| Component | Role |
|-----------|------|
| `rootName` | Base directory where all product images are stored |
| `cacheManager` | Dependency that supplies `rootName` via `CMSManager` |
| `addProductImage` | Persists an `ImageContentFile` to the appropriate sub‑directory |
| `removeProductImage`, `removeProductImages`, `removeImages` | Delete files or directories for a specific product or merchant |
| `buildRootPath` | Helper that builds the root path string |
| `createDirectoryIfNorExist` | Utility that ensures a directory exists before writing |

**Design patterns / frameworks**  

* **Singleton** – `getInstance()` creates a single instance of the manager.  
* **Dependency Injection** – `cacheManager` is injected (Spring’s `@PostConstruct`).  
* **Java NIO** – `Path`, `Files`, `StandardCopyOption` for file operations.  
* **SLF4J** – Logging.  

---

## 2. Detailed Description  

### Execution Flow

1. **Initialisation**  
   * After the bean is constructed, `@PostConstruct` invokes `init()`.  
   * `init()` casts the injected `cacheManager` to `CMSManager` and pulls the `rootName` from it.  
   * The singleton instance (`fileManager`) can also be retrieved via `getInstance()`.

2. **Adding an Image** (`addProductImage`)  
   * Build a path of the form  
     ```
     <rootName>/products/<merchantCode>/<productSku>/<size>/<fileName>
     ```  
   * Ensure every directory in the path exists (via `createDirectoryIfNorExist`).  
   * Copy the file from `ImageContentFile.getFile()` to the constructed path.  
   * Exceptions are wrapped in a `ServiceException`.

3. **Removing Images**  
   * **All images of a merchant** – `removeImages(String merchantStoreCode)` deletes the merchant’s top‑level directory.  
   * **All images of a product** – `removeProductImages(Product)` deletes the product folder.  
   * **Single image** – `removeProductImage(ProductImage)` deletes both the *SMALL* and *LARGE* copies of the file.  
   * All deletions use `Files.deleteIfExists`, which fails silently if the path is a non‑empty directory.

4. **Retrieval methods**  
   * The class declares several `get…` methods, but all currently return `null`.  
   * The intended behaviour (as noted in the comments) is that a web server will serve the files directly, so the Java layer may not need to stream them.

### Assumptions & Constraints

| Item | Description |
|------|-------------|
| **File system** | The code assumes a POSIX‑style file system with `/` separators. `Constants.SLASH` is hardcoded. |
| **Directory structure** | All product images are stored under `<root>/products/...`. No versioning or cache busting. |
| **Concurrency** | The singleton is *not* thread‑safe. Multiple threads could instantiate it concurrently. |
| **Dependency** | `cacheManager` must be non‑null and implement `CMSManager`. |
| **Deletion** | Deleting a non‑empty directory will throw `DirectoryNotEmptyException`. No recursive cleanup is performed. |
| **Resource handling** | InputStreams are never closed – potential leaks. |
| **Error handling** | All exceptions are wrapped in `ServiceException` without distinguishing causes. |

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `getInstance()` | Singleton access | – | `CmsImageFileManagerImpl` | None | Not thread‑safe. |
| `init()` | Sets `rootName` from injected `cacheManager` | – | – | Logs initialisation | Invoked via `@PostConstruct`. |
| `addProductImage(ProductImage, ImageContentFile)` | Persist an image file | `productImage`, `contentImage` | – | Writes file to disk | Creates directories, copies file, may leak `InputStream`. |
| `removeImages(String)` | Delete all images for a merchant | `merchantStoreCode` | – | Deletes directory | Fails if directory not empty. |
| `removeProductImages(Product)` | Delete all images for a product | `product` | – | Deletes directory | Same limitation as above. |
| `removeProductImage(ProductImage)` | Delete both small & large copies | `productImage` | – | Deletes files | Uses absolute path building. |
| `getProductImage(ProductImage)` | **Stub** – intended to retrieve a product image | `productImage` | `OutputContentFile` | None | Returns `null`. |
| `getImages(MerchantStore, FileContentType)` | **Stub** – retrieve images by store & type | `store`, `imageContentType` | `List<OutputContentFile>` | None | Returns `null`. |
| `getImages(Product)` | **Stub** – retrieve images for a product | `product` | `List<OutputContentFile>` | None | Returns `null`. |
| `getImages(String, FileContentType)` | **Stub** – retrieve images by store code & type | `merchantStoreCode`, `imageContentType` | `List<OutputContentFile>` | None | Returns `null`. |
| `getProductImage(String, String, String)` | Convenience wrapper for `size = SMALL` | `merchantStoreCode`, `productCode`, `imageName` | `OutputContentFile` | None | Delegates to private overload. |
| `getProductImage(String, String, String, ProductImageSize)` | Convenience wrapper for `size = size.name()` | `merchantStoreCode`, `productCode`, `imageName`, `size` | `OutputContentFile` | None | Delegates to private overload. |
| `getProductImage(String, String, String, String)` | **Stub** – intended to resolve a path | `merchantStoreCode`, `productCode`, `imageName`, `size` | `OutputContentFile` | None | Returns `null`. |
| `buildRootPath()` | Helper to construct `<root>/products/` | – | `String` | None | Uses `getRootName()`; may produce relative paths. |
| `createDirectoryIfNorExist(Path)` | Ensure directory exists | `path` | – | Creates dir if missing | Uses `Files.createDirectory` (not recursive). |
| `setRootName(String)`, `getRootName()` | Accessors | – | – | – | |
| `setCacheManager(LocalCacheManagerImpl)`, `getCacheManager()` | Accessors | – | – | – | |

---

## 4. Dependencies  

| Library / Framework | Role | Standard / Third‑Party |
|---------------------|------|------------------------|
| `java.io.*`, `java.nio.file.*` | File I/O, directory handling | Standard |
| `org.slf4j.*` | Logging | Third‑party (SLF4J) |
| `com.salesmanager.core.*` | Domain model (Product, MerchantStore, etc.) and custom exceptions | Project‑specific |
| `javax.annotation.PostConstruct` | Lifecycle hook for Spring/Java EE | Standard / JSR-250 |
| `java.util.List` | Collections | Standard |

No external web server or cloud storage SDKs are used; all interactions are local file system based.

---

## 5. Additional Notes & Recommendations  

### 5.1 Correctness & Robustness  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| `createDirectoryIfNorExist` uses `Files.createDirectory` | Fails when parent dirs missing (e.g., root not created yet). | Use `Files.createDirectories(path)` (recursive). |
| `removeImages` / `removeProductImages` delete non‑empty dirs | Throws `DirectoryNotEmptyException`. | Use `Files.walkFileTree` or Apache Commons `FileUtils.deleteDirectory`. |
| `InputStream` not closed | File descriptor leak. | Wrap in try‑with‑resources (`try (InputStream is = contentImage.getFile()) { … }`). |
| Singleton not thread‑safe | Race condition on first call. | Synchronize `getInstance()` or use eager init / `volatile` double‑checked locking. |
| `serialVersionUID` present but class doesn’t implement `Serializable` | Unnecessary. | Remove field. |
| Hardcoded slash (`Constants.SLASH`) | Platform‑independent path building broken on Windows. | Use `File.separator` or `Path` operations (`path.resolve()`). |
| `removeProductImage` deletes small & large copies separately but doesn't check if file exists | Silent failures; not atomic. | Add existence check or log missing files. |
| `getProductImage` / `getImages` all return `null` | Incomplete implementation. | Either fully implement or throw `UnsupportedOperationException`. |
| `cacheManager` cast to `CMSManager` | Risk of `ClassCastException`. | Enforce type in setter or use `instanceof`. |
| No validation of `FileContentType` | Potential `NullPointerException`. | Guard against null and use enum comparison (`==`). |
| `rootName` default `""` | Path may be relative; unpredictable base. | Enforce absolute path in constructor or config. |

### 5.2 Design Enhancements  

1. **Decouple from Singleton** – let Spring manage bean lifecycle; remove static `getInstance()` or replace with `@Component` + `@Autowired`.  
2. **Implement Retrieval** – The stubbed `get…` methods should return an `OutputContentFile` pointing to the file or stream, or throw an exception if not supported.  
3. **Use Path APIs** – Replace all manual string concatenation with `Path.resolve()` to avoid path errors.  
4. **Error Handling** – Distinguish between I/O errors, missing files, and permission issues; wrap with specific `ServiceException` messages.  
5. **Unit Tests** – Add tests for directory creation, file copy, and deletion logic, using temporary directories.  

### 5.3 Performance & Scalability  

* Writing large images synchronously on the same thread may block the application; consider async uploads or streaming.  
* Deleting directories recursively can be expensive; if many images, batch deletion or use OS‑level `rm -rf`.  

### 5.4 Security  

* Validate `productImage.getProductImage()` and `contentImage.getFileName()` to prevent path traversal (`..`).  
* Ensure the root directory is not world‑writeable; set proper file permissions.  

---

**Conclusion**  
`CmsImageFileManagerImpl` provides a straightforward local‑file implementation of image storage for products.  It functions correctly for the simple “add” operation but lacks robust error handling, directory cleanup, and complete retrieval logic.  Addressing the points above will make the class safer, more maintainable, and ready for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.product.local;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.List;
import javax.annotation.PostConstruct;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.business.modules.cms.impl.LocalCacheManagerImpl;
import com.salesmanager.core.business.modules.cms.product.ProductAssetsManager;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;

/**
 * Manager for storing and deleting image files from the CMS which is a web server
 * 
 * Manages - Product images
 * 
 * @author Carl Samson
 */
public class CmsImageFileManagerImpl
    implements ProductAssetsManager {

  /**
   * 
   */
  private static final long serialVersionUID = 1L;

  private static final Logger LOGGER = LoggerFactory.getLogger(CmsImageFileManagerImpl.class);

  private static CmsImageFileManagerImpl fileManager = null;

  private final static String ROOT_NAME = "";

  private final static String SMALL = "SMALL";
  private final static String LARGE = "LARGE";

  private static final String ROOT_CONTAINER = "products";

  private String rootName = ROOT_NAME;

  private LocalCacheManagerImpl cacheManager;

  @PostConstruct
  void init() {

    this.rootName = ((CMSManager) cacheManager).getRootName();
    LOGGER.info("init " + getClass().getName() + " setting root" + this.rootName);

  }

  public static CmsImageFileManagerImpl getInstance() {

    if (fileManager == null) {
      fileManager = new CmsImageFileManagerImpl();
    }

    return fileManager;

  }

  private CmsImageFileManagerImpl() {

  }

  /**
   * root/products/<merchant id>/<product id>/1.jpg
   */

  @Override
  public void addProductImage(ProductImage productImage, ImageContentFile contentImage)
      throws ServiceException {


    try {

      // base path
      String rootPath = this.buildRootPath();
      Path confDir = Paths.get(rootPath);
      this.createDirectoryIfNorExist(confDir);

      // node path
      StringBuilder nodePath = new StringBuilder();
      nodePath.append(rootPath).append(productImage.getProduct().getMerchantStore().getCode());
      Path merchantPath = Paths.get(nodePath.toString());
      this.createDirectoryIfNorExist(merchantPath);

      // product path
      nodePath.append(Constants.SLASH).append(productImage.getProduct().getSku())
          .append(Constants.SLASH);
      Path dirPath = Paths.get(nodePath.toString());
      this.createDirectoryIfNorExist(dirPath);

      // small large
      if (contentImage.getFileContentType().name().equals(FileContentType.PRODUCT.name())) {
        nodePath.append(SMALL);
      } else if (contentImage.getFileContentType().name()
          .equals(FileContentType.PRODUCTLG.name())) {
        nodePath.append(LARGE);
      }
      Path sizePath = Paths.get(nodePath.toString());
      this.createDirectoryIfNorExist(sizePath);


      // file creation
      nodePath.append(Constants.SLASH).append(contentImage.getFileName());


      Path path = Paths.get(nodePath.toString());
      InputStream isFile = contentImage.getFile();

      Files.copy(isFile, path, StandardCopyOption.REPLACE_EXISTING);


    } catch (Exception e) {

      throw new ServiceException(e);

    }

  }

  @Override
  public OutputContentFile getProductImage(ProductImage productImage) throws ServiceException {

    // the web server takes care of the images
    return null;

  }


  public List<OutputContentFile> getImages(MerchantStore store, FileContentType imageContentType)
      throws ServiceException {

    // the web server takes care of the images

    return null;

  }

  @Override
  public List<OutputContentFile> getImages(Product product) throws ServiceException {

    // the web server takes care of the images

    return null;
  }



  @Override
  public void removeImages(final String merchantStoreCode) throws ServiceException {

    try {


      StringBuilder merchantPath = new StringBuilder();
      merchantPath.append(buildRootPath()).append(Constants.SLASH).append(merchantStoreCode);

      Path path = Paths.get(merchantPath.toString());

      Files.deleteIfExists(path);


    } catch (Exception e) {
      throw new ServiceException(e);
    }


  }


  @Override
  public void removeProductImage(ProductImage productImage) throws ServiceException {


    try {


      StringBuilder nodePath = new StringBuilder();
      nodePath.append(buildRootPath()).append(Constants.SLASH)
          .append(productImage.getProduct().getMerchantStore().getCode()).append(Constants.SLASH)
          .append(productImage.getProduct().getSku());

      // delete small
      StringBuilder smallPath = new StringBuilder(nodePath);
      smallPath.append(Constants.SLASH).append(SMALL).append(Constants.SLASH)
          .append(productImage.getProductImage());


      Path path = Paths.get(smallPath.toString());

      Files.deleteIfExists(path);

      // delete large
      StringBuilder largePath = new StringBuilder(nodePath);
      largePath.append(Constants.SLASH).append(LARGE).append(Constants.SLASH)
          .append(productImage.getProductImage());


      path = Paths.get(largePath.toString());

      Files.deleteIfExists(path);

    } catch (Exception e) {
      throw new ServiceException(e);
    }


  }

  @Override
  public void removeProductImages(Product product) throws ServiceException {

    try {


      StringBuilder nodePath = new StringBuilder();
      nodePath.append(buildRootPath()).append(Constants.SLASH)
          .append(product.getMerchantStore().getCode()).append(Constants.SLASH)
          .append(product.getSku());


      Path path = Paths.get(nodePath.toString());

      Files.deleteIfExists(path);

    } catch (Exception e) {
      throw new ServiceException(e);
    }

  }


  @Override
  public List<OutputContentFile> getImages(final String merchantStoreCode,
      FileContentType imageContentType) throws ServiceException {

    // the web server taks care of the images

    return null;
  }

  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName) throws ServiceException {
    return getProductImage(merchantStoreCode, productCode, imageName,
        ProductImageSize.SMALL.name());
  }

  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName, ProductImageSize size) throws ServiceException {
    return getProductImage(merchantStoreCode, productCode, imageName, size.name());
  }

  private OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName, String size) throws ServiceException {

    return null;

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

  public void setRootName(String rootName) {
    this.rootName = rootName;
  }

  public String getRootName() {
    return rootName;
  }

  public LocalCacheManagerImpl getCacheManager() {
    return cacheManager;
  }

  public void setCacheManager(LocalCacheManagerImpl cacheManager) {
    this.cacheManager = cacheManager;
  }



}



```
