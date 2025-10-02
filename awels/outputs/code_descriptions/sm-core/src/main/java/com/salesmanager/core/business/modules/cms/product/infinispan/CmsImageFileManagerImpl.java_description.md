# CmsImageFileManagerImpl.java

## Review

## 1. Summary

`CmsImageFileManagerImpl` is a Spring‑managed (although it also implements a classic singleton pattern) component that stores and retrieves product images in an **Infinispan Tree Cache**.  
Images are keyed by a path of the form  

```
<merchant‑code>/<product‑sku>/<size>/<file‑name>
```

where `<size>` is either `SMALL` or `LARGE`.  The manager implements `ProductAssetsManager`, exposing CRUD operations for product images and batch operations that operate on entire merchants or products.

Key design points  
- **Tree Cache**: Leverages the hierarchical API of Infinispan to organize image data.  
- **Singleton + DI**: Uses a static `getInstance()` but also relies on `@PostConstruct`/`setCacheManager` for wiring, which is contradictory.  
- **File handling**: Reads whole files into memory as `byte[]` and stores those blobs in the cache.  
- **Content‑type detection**: Uses `URLConnection.getFileNameMap()` to infer MIME types from filenames.

---

## 2. Detailed Description

### Architecture

| Layer | Responsibility |
|-------|----------------|
| **CacheManager** | Holds the Infinispan `TreeCache` instance and its configuration (root name, etc.). |
| **CmsImageFileManagerImpl** | CRUD façade over the cache: add, retrieve, list, and delete images. |
| **ProductAssetsManager (interface)** | Declares the contract that this implementation satisfies. |

### Execution Flow

1. **Startup**  
   - Spring injects a `CacheManager` via `setCacheManager`.  
   - `@PostConstruct` runs `init()`, which sets the local `rootName` from the manager.

2. **Adding an Image** (`addProductImage`)  
   - Builds the node path (merchant → product → size).  
   - Reads the image `InputStream` fully into a `byte[]`.  
   - Stores the array in the corresponding `Node` with the image filename as the key.

3. **Retrieving an Image** (`getProductImage` overloads)  
   - Resolves the node path, fetches the byte array, wraps it in an `OutputContentFile`, and returns it.  
   - MIME type is guessed via `URLConnection.getFileNameMap()`.

4. **Listing Images** (`getImages` variants)  
   - Traverses the appropriate node hierarchy.  
   - For each key found, reads the byte array and constructs an `OutputContentFile`.  
   - Returns a list of such objects.

5. **Removing Images** (`removeProductImage`, `removeProductImages`, `removeImages`)  
   - Builds the relevant node path and calls `remove()` on the node or the root.  

6. **Shutdown** (`stopFileManager`)  
   - Stops the underlying Infinispan manager.

### Assumptions & Constraints

- All image data fits comfortably in memory; no streaming to/from disk.  
- Image files are small enough that a single byte array per image is acceptable.  
- The cache is configured for persistence or replication elsewhere; this class does not handle those concerns.  
- The `FileContentType` enum distinguishes between regular (`PRODUCT`) and large (`PRODUCTLG`) images.  
- The `ProductImageSize` enum is used for size look‑ups but only contains `SMALL` in the current code base.

---

## 3. Functions / Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `stopFileManager()` | Gracefully stops the Infinispan engine | – | – | Calls `cacheManager.getManager().stop()` |
| `init()` | Post‑construction init | – | – | Sets `rootName` from `CacheManager` |
| `getInstance()` | Classic singleton factory | – | `CmsImageFileManagerImpl` | Lazily creates a singleton; **not thread‑safe** |
| `addProductImage(ProductImage, ImageContentFile)` | Persist a product image | `ProductImage`, `ImageContentFile` | – | Writes bytes to the cache |
| `getProductImage(ProductImage)` | Retrieve a specific image by `ProductImage` object | `ProductImage` | `OutputContentFile` | Reads bytes from cache |
| `getImages(MerchantStore, FileContentType)` | Delegate to string‑based version | `MerchantStore`, `FileContentType` | `List<OutputContentFile>` | – |
| `getImages(Product)` | List all images for a product | `Product` | `List<OutputContentFile>` | Reads entire product node |
| `removeImages(String)` | Delete all images for a merchant | merchant code | – | Removes merchant node |
| `removeProductImage(ProductImage)` | Delete a single image | `ProductImage` | – | Removes key from product node |
| `removeProductImages(Product)` | Delete all images for a product | `Product` | – | Removes product node |
| `getImages(String, FileContentType)` | List images for a merchant (unimplemented size filtering) | merchant code, content type | `List<OutputContentFile>` | Traverses children nodes |
| `getProductImage(String, String, String)` | Retrieve by merchant/product/image name, default size | merchant code, product code, image name | `OutputContentFile` | Calls overloaded method |
| `getProductImage(String, String, String, ProductImageSize)` | Same as above but with size enum | merchant code, product code, image name, size | `OutputContentFile` | |
| `getProductImage(String, String, String, String)` | Core implementation that fetches bytes | merchant code, product code, image name, size string | `OutputContentFile` | Reads from cache |
| `getNode(String)` | Lazily obtain/create a node in the tree cache | node path string | `Node<String,Object>` | Creates new node if missing |
| `setCacheManager(CacheManager)` | Dependency injection setter | `CacheManager` | – | Sets internal reference |
| `setRootName(String)` | Override root name | root name | – | |
| `getRootName()` | Return root name | – | `String` | – |

---

## 4. Dependencies

| Library | Purpose | Type |
|---------|---------|------|
| **Infinispan (TreeCache API)** | Hierarchical cache storage | Third‑party |
| **SLF4J** | Logging | Third‑party |
| **Apache Commons IO** (`IOUtils`) | Stream copy utilities | Third‑party |
| **Java SE** | `FileNameMap`, `URLConnection`, `ByteArrayInputStream`, etc. | Standard |

The code does not use any external frameworks beyond Infinispan and SLF4J; however, the presence of `@PostConstruct` and the usage of setter injection indicates that a Spring or CDI container is expected.

---

## 5. Additional Notes & Recommendations

### Design Issues

1. **Singleton vs. DI**  
   - `getInstance()` creates a static singleton that **does not** have `cacheManager` injected.  
   - In a Spring environment, beans should be instantiated by the container; the static method is unnecessary and can lead to `NullPointerException` if used outside the container.  
   - **Fix**: Remove `fileManager`/`getInstance()` and let Spring manage the bean scope.

2. **Thread Safety of `getNode`**  
   - Two concurrent threads could both find `nd == null` and attempt to `addChild`.  
   - This could result in duplicate nodes or race conditions.  
   - **Fix**: Use `cacheManager.getTreeCache().getRoot().getOrCreateChild(contentFilesFqn)` if available, or synchronize on the root.

3. **Inconsistent Return Policies**  
   - `getImages(Product)` returns `null` when no node is found, while other list methods return an empty list.  
   - This inconsistency can lead to `NullPointerException` for callers.  
   - **Fix**: Always return an empty list.

4. **Incorrect Use of `merchantNode.get(key)` in `getImages(String, FileContentType)`**  
   - The method should retrieve the byte array from the **child node**, not the parent merchant node.  
   - **Fix**: Replace `merchantNode.get(key)` with `node.get(key)` inside the nested loop.

5. **Resource Leaks**  
   - `InputStream` from `contentImage.getFile()` is never closed in `addProductImage`.  
   - In `getProductImage`, the stream is closed only in the `finally` block if not `null`; however, if `imageBytes` is `null` the method returns early without closing the stream.  
   - **Fix**: Use try‑with‑resources for all stream handling.

6. **Content‑Type Detection**  
   - `URLConnection.getFileNameMap()` relies solely on the file extension and is not reliable for all image types.  
   - Consider using `Files.probeContentType(Path)` or a third‑party library like Apache Tika.

7. **Hard‑coded Size Strings**  
   - The implementation uses literal strings `"SMALL"` and `"LARGE"`.  
   - A better approach is to expose a `ProductImageSize` enum and map it to the tree path.  
   - **Fix**: Use the enum consistently throughout.

8. **Error Handling**  
   - Catching generic `Exception` and re‑throwing as `ServiceException` masks the root cause.  
   - Consider catching only the expected checked exceptions or logging the original exception details.

9. **Missing Transactional Support**  
   - If Infinispan is configured in a transactional mode, the operations should be wrapped in transactions to guarantee atomicity.  
   - The current implementation assumes each cache operation is atomic.

10. **Performance Considerations**  
    - Storing entire image files in memory may be fine for small images but will not scale.  
    - If large images or many images are expected, consider streaming the data directly to a persistent store or a blob container.

### Suggested Refactoring Steps

1. **Remove the static singleton** – rely on Spring bean lifecycle.  
2. **Introduce a thread‑safe node creation** – e.g., `getOrCreateNode`.  
3. **Fix the `getImages` traversal bug** – use child node for key retrieval.  
4. **Standardise return values** – always return an empty list, never `null`.  
5. **Close all streams** – use try‑with‑resources.  
6. **Replace MIME detection** – use a robust library.  
7. **Add unit tests** – cover all CRUD paths and edge cases.  
8. **Document the node path structure** – include diagrams in Javadoc.  
9. **Handle configuration** – expose root name and cache manager via Spring properties.  
10. **Add metrics** – track cache hit/miss rates for image retrieval.

### Edge Cases Not Handled

- **Concurrent writes** to the same image may overwrite each other silently.  
- **Missing image file** (`null` stream) in `addProductImage` – will throw a `NullPointerException`.  
- **Large file sizes** will cause `OutOfMemoryError`.  
- **Invalid size string** in `getProductImage` – no validation, may return wrong path.  
- **Cache eviction** – no policy; if cache reaches its max size, images may be evicted unexpectedly.

---

**Conclusion**:  
`CmsImageFileManagerImpl` provides the core functionality for storing and retrieving product images in Infinispan but suffers from design inconsistencies, potential concurrency bugs, and resource leaks. A refactor that aligns the component with standard Spring bean practices, enforces thread safety, and cleans up stream handling would greatly improve reliability and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.product.infinispan;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.net.FileNameMap;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Set;
import javax.annotation.PostConstruct;
import org.apache.commons.io.IOUtils;
import org.infinispan.tree.Fqn;
import org.infinispan.tree.Node;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.impl.CMSManager;
import com.salesmanager.core.business.modules.cms.impl.CacheManager;
import com.salesmanager.core.business.modules.cms.product.ProductAssetsManager;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;

/**
 * Manager for storing in retrieving image files from the CMS This is a layer on top of Infinispan
 * https://docs.jboss.org/author/display/ISPN/Tree+API+Module
 * 
 * Manages - Product images
 * 
 * @author Carl Samson
 */
public class CmsImageFileManagerImpl implements ProductAssetsManager {

  /**
   * 
   */
  private static final long serialVersionUID = 1L;

  private static final Logger LOGGER = LoggerFactory.getLogger(CmsImageFileManagerImpl.class);

  private static CmsImageFileManagerImpl fileManager = null;

  private final static String ROOT_NAME = "product-merchant";

  private final static String SMALL = "SMALL";
  private final static String LARGE = "LARGE";

  private String rootName = ROOT_NAME;

  private CacheManager cacheManager;


  /**
   * Requires to stop the engine when image servlet un-deploys
   */
  public void stopFileManager() {

    try {
      LOGGER.info("Stopping CMS");
      cacheManager.getManager().stop();
    } catch (Exception e) {
      LOGGER.error("Error while stopping CmsImageFileManager", e);
    }
  }

  @PostConstruct
  void init() {

    this.rootName = cacheManager.getRootName();
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
   * root -productFiles -merchant-id PRODUCT-ID(key) -> CacheAttribute(value) - image 1 - image 2 -
   * image 3
   */

  @Override
  public void addProductImage(ProductImage productImage, ImageContentFile contentImage)
      throws ServiceException {

    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }

    try {

      // node
      StringBuilder nodePath = new StringBuilder();
      nodePath.append(productImage.getProduct().getMerchantStore().getCode())
          .append(Constants.SLASH).append(productImage.getProduct().getSku())
          .append(Constants.SLASH);


      if (contentImage.getFileContentType().name().equals(FileContentType.PRODUCT.name())) {
        nodePath.append(SMALL);
      } else if (contentImage.getFileContentType().name()
          .equals(FileContentType.PRODUCTLG.name())) {
        nodePath.append(LARGE);
      }

      Node<String, Object> productNode = this.getNode(nodePath.toString());


      InputStream isFile = contentImage.getFile();

      ByteArrayOutputStream output = new ByteArrayOutputStream();
      IOUtils.copy(isFile, output);


      // object for a given product containing all images
      productNode.put(contentImage.getFileName(), output.toByteArray());



    } catch (Exception e) {

      throw new ServiceException(e);

    }

  }

  @Override
  public OutputContentFile getProductImage(ProductImage productImage) throws ServiceException {

    return getProductImage(productImage.getProduct().getMerchantStore().getCode(),
        productImage.getProduct().getSku(), productImage.getProductImage());

  }


  public List<OutputContentFile> getImages(MerchantStore store, FileContentType imageContentType)
      throws ServiceException {

    return getImages(store.getCode(), imageContentType);

  }

  @Override
  public List<OutputContentFile> getImages(Product product) throws ServiceException {

    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }

    List<OutputContentFile> images = new ArrayList<OutputContentFile>();


    try {


      FileNameMap fileNameMap = URLConnection.getFileNameMap();
      StringBuilder nodePath = new StringBuilder();
      nodePath.append(product.getMerchantStore().getCode());

      Node<String, Object> merchantNode = this.getNode(nodePath.toString());

      if (merchantNode == null) {
        return null;
      }


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


    }

    catch (Exception e) {
      throw new ServiceException(e);
    } finally {

    }

    return images;
  }



  @SuppressWarnings("unchecked")
  @Override
  public void removeImages(final String merchantStoreCode) throws ServiceException {
    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }

    try {


      final StringBuilder merchantPath = new StringBuilder();
      merchantPath.append(getRootName()).append(merchantStoreCode);
      cacheManager.getTreeCache().getRoot().remove(merchantPath.toString());



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {

    }

  }


  @Override
  public void removeProductImage(ProductImage productImage) throws ServiceException {

    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }

    try {


      StringBuilder nodePath = new StringBuilder();
      nodePath.append(productImage.getProduct().getMerchantStore().getCode())
          .append(Constants.SLASH).append(productImage.getProduct().getSku());


      Node<String, Object> productNode = this.getNode(nodePath.toString());
      productNode.remove(productImage.getProductImage());



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {

    }

  }

  @Override
  public void removeProductImages(Product product) throws ServiceException {

    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }

    try {


      StringBuilder nodePath = new StringBuilder();
      nodePath.append(product.getMerchantStore().getCode());


      Node<String, Object> merchantNode = this.getNode(nodePath.toString());

      merchantNode.remove(product.getSku());



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {

    }

  }


  @Override
  public List<OutputContentFile> getImages(final String merchantStoreCode,
      FileContentType imageContentType) throws ServiceException {
    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }
    List<OutputContentFile> images = new ArrayList<OutputContentFile>();
    FileNameMap fileNameMap = URLConnection.getFileNameMap();

    try {


      StringBuilder nodePath = new StringBuilder();
      nodePath.append(merchantStoreCode);


      Node<String, Object> merchantNode = this.getNode(nodePath.toString());

      Set<Node<String, Object>> childs = merchantNode.getChildren();

      // TODO image sizes
      for (Node<String, Object> node : childs) {

        for (String key : node.getKeys()) {


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

      }



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {

    }

    return images;
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

    if (cacheManager.getTreeCache() == null) {
      throw new ServiceException(
          "CmsImageFileManagerInfinispan has a null cacheManager.getTreeCache()");
    }
    InputStream input = null;
    OutputContentFile contentImage = new OutputContentFile();
    try {

      FileNameMap fileNameMap = URLConnection.getFileNameMap();

      // SMALL by default
      StringBuilder nodePath = new StringBuilder();
      nodePath.append(merchantStoreCode).append(Constants.SLASH).append(productCode)
          .append(Constants.SLASH).append(size);

      Node<String, Object> productNode = this.getNode(nodePath.toString());


      byte[] imageBytes = (byte[]) productNode.get(imageName);

      if (imageBytes == null) {
        LOGGER.warn("Image " + imageName + " does not exist");
        return null;// no post processing will occur
      }

      input = new ByteArrayInputStream(imageBytes);
      ByteArrayOutputStream output = new ByteArrayOutputStream();
      IOUtils.copy(input, output);

      String contentType = fileNameMap.getContentTypeFor(imageName);

      contentImage.setFile(output);
      contentImage.setMimeType(contentType);
      contentImage.setFileName(imageName);



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {
      if (input != null) {
        try {
          input.close();
        } catch (Exception ignore) {
        }
      }
    }

    return contentImage;

  }


  @SuppressWarnings("unchecked")
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

  public CacheManager getCacheManager() {
    return cacheManager;
  }

  public void setCacheManager(CacheManager cacheManager) {
    this.cacheManager = cacheManager;
  }

  public void setRootName(String rootName) {
    this.rootName = rootName;
  }

  public String getRootName() {
    return rootName;
  }



}



```
