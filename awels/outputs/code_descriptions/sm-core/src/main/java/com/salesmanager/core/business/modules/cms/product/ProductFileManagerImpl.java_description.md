# ProductFileManagerImpl.java

## Review

## 1. Summary
The `ProductFileManagerImpl` class implements the concrete business logic for handling product images in the SalesManager platform.  
* **Purpose** – It manages the lifecycle of product images: uploading (with optional resizing/cropping), retrieving (original or resized), and deleting images for a given product or store.  
* **Key components** –  
  * **`uploadImage`** (`ProductImagePut`): Persists an image to the underlying storage.  
  * **`getImage`** (`ProductImageGet`): Reads images back from storage.  
  * **`removeImage`** (`ProductImageRemove`): Deletes images from storage.  
  * **`configuration`** (`CoreConfiguration`): Supplies runtime properties such as target image dimensions and crop‑enable flag.  
* **Design patterns / frameworks** –  
  * **Dependency Injection** – All three CRUD collaborators are injected via setters.  
  * **Strategy/Facade** – The class exposes a unified API (`addProductImage`, `getProductImage`, `removeProductImage`, etc.) while delegating the actual persistence logic to the injected collaborators.  
  * **Utility classes** – `ProductImageCropUtils` and `ProductImageSizeUtils` encapsulate image manipulation logic.

## 2. Detailed Description
1. **Initialization** – The class relies on an external `CoreConfiguration` instance. No heavy initialization logic is present; the class is effectively a thin service wrapper.  
2. **Runtime behavior** –  
   * **Adding an image** (`addProductImage`)  
     * Reads the entire file into a byte array to create two `ByteArrayInputStream` instances – one for the original image, one for processing.  
     * Decodes the image with `ImageIO.read`.  
     * Uploads the original image as `PRODUCTLG`.  
     * Optionally resizes the image to the dimensions specified by `PRODUCT_IMAGE_HEIGHT_SIZE` / `PRODUCT_IMAGE_WIDTH_SIZE`.  
       * Cropping is performed if `CROP_UPLOADED_IMAGES` is `true`.  
       * Resizing uses `ProductImageSizeUtils.resizeWithRatio`.  
     * Stores the resized image as a temporary file and uploads it as `PRODUCT`.  
     * The small‑size logic is commented out – currently the system only stores the original and the “large” version.  
   * **Retrieving images** – The class delegates to `ProductImageGet` for all retrieval operations, always returning the original image unless a specific size is requested.  
   * **Deleting images** – Delegated to `ProductImageRemove`. The code contains commented‑out logic for removing derived images (large/small prefixes), but currently only removes the record that was passed in.  
3. **Cleanup** – The only cleanup performed is the closing of the original image stream (`productImage.getImage()`). No explicit closing of the other streams or the temporary file is guaranteed if an exception occurs during upload.

## 3. Functions / Methods
| Method | Purpose | Parameters | Return | Side‑Effects / Notes |
|--------|---------|------------|--------|----------------------|
| `addProductImage(ProductImage, ImageContentFile)` | Handles the complete lifecycle of uploading an image: read, optionally crop/resize, and store original & resized copies. | `productImage`, `contentImage` | void | Throws `ServiceException` on any error; deletes temp file after upload; closes original image stream in finally block. |
| `getProductImage(ProductImage)` | Returns the original image for the supplied `ProductImage`. | `productImage` | `OutputContentFile` | Delegates to `getImage`. |
| `getProductImage(String, String, String)` | Returns a specific image by merchant store, product code and image name. | `merchantStoreCode`, `productCode`, `imageName` | `OutputContentFile` | Delegates to `getImage`. |
| `getProductImage(String, String, String, ProductImageSize)` | Same as above but with a requested size. | `merchantStoreCode`, `productCode`, `imageName`, `size` | `OutputContentFile` | Delegates to `getImage`. |
| `getImages(String, FileContentType)` | Retrieves all images for a store and content type. Currently ignores the passed `imageContentType` and always uses `FileContentType.PRODUCT`. | `merchantStoreCode`, `imageContentType` | `List<OutputContentFile>` | Delegates to `getImage`. |
| `getImages(Product)` | Retrieves all images for a product. | `product` | `List<OutputContentFile>` | Delegates to `getImage`. |
| `removeProductImage(ProductImage)` | Deletes a single image record. | `productImage` | void | Delegates to `removeImage`. |
| `removeProductImages(Product)` | Deletes all images for a product. | `product` | void | Delegates to `removeImage`. |
| `removeImages(String)` | Deletes all images for a store. | `merchantStoreCode` | void | Delegates to `removeImage`. |
| `setConfiguration(CoreConfiguration)` / `getConfiguration()` | Setter/getter for config. | - | - | - |
| `setUploadImage(ProductImagePut)` / `getUploadImage()` | Setter/getter for uploader. | - | - | - |
| `setGetImage(ProductImageGet)` / `getGetImage()` | Setter/getter for reader. | - | - | - |
| `setRemoveImage(ProductImageRemove)` / `getRemoveImage()` | Setter/getter for remover. | - | - | - |

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang3.StringUtils` | Third‑party | Apache Commons Lang |
| `org.slf4j.Logger` / `LoggerFactory` | Third‑party | SLF4J logging façade |
| `java.awt.image.BufferedImage`, `javax.imageio.ImageIO` | Standard | Image handling |
| `java.net.URLConnection` / `FileNameMap` | Standard | MIME type detection |
| `java.io.*`, `java.nio.file` | Standard | File & stream IO |
| `com.salesmanager.core.*` | Internal | Core business models, exceptions, utilities |

No platform‑specific APIs are used; the code should run on any JVM that supports `javax.imageio`.

## 5. Additional Notes & Recommendations
### 5.1 Edge Cases & Error Handling
1. **Null or malformed parameters** – None of the public methods guard against `null` inputs or malformed `productImage` objects. Adding defensive checks or documenting that callers must validate is advisable.  
2. **Missing/invalid configuration** – `Integer.parseInt` on `PRODUCT_IMAGE_HEIGHT_SIZE` / `PRODUCT_IMAGE_WIDTH_SIZE` will throw `NumberFormatException` if the properties are missing or non‑numeric. The current code catches it as a generic `Exception` and re‑wraps it in `ServiceException`, but a more explicit validation step would improve clarity.  
3. **Stream leakage** – `is1` and `is2` are never closed; `ByteArrayInputStream` does not require explicit closing, but it’s still good practice to use try‑with‑resources for all streams.  
4. **Temporary file cleanup** – If `uploadImage.addProductImage` throws an exception, `tempLarge.delete()` is never executed. Consider using a `finally` block or a `try‑with‑resources` that deletes the file in a `File.deleteOnExit()` fallback.  

### 5.2 Code Quality & Maintainability
1. **Hard‑coded file extensions** – The fallback to `"jpeg"` may not match the actual file type if the MIME type detection fails. A more robust mapping or use of `ImageIO.getImageReaders` would be safer.  
2. **Redundant string building** – `new StringBuilder().append(productImage.getProduct().getId()).append("tmpLarge")` can be replaced with `String.format("%s_tmpLarge", productImage.getProduct().getId())` for readability.  
3. **Commented code** – Large blocks of commented‑out logic for small‑size images and for prefixing image names should be removed or refactored into a configurable feature.  
4. **Method overloading** – The `getImages(String, FileContentType)` ignores its second argument. Either support the argument or remove it from the signature to avoid confusion.  
5. **Magic strings** – Constants like `"L-"` and `"S-"` prefixes are hard‑coded; they should be defined as `static final` fields if used elsewhere.  

### 5.3 Potential Enhancements
1. **Batch processing** – Allow uploading multiple images in a single call to reduce I/O overhead.  
2. **Streaming uploads** – Avoid materializing the entire image into memory; process the stream directly with `ImageIO.createImageInputStream`.  
3. **Configuration validation** – Validate all image size properties at startup and expose clear error messages.  
4. **Image type detection** – Use `ImageIO.getImageReaders` to determine format rather than MIME‑type guess.  
5. **Unit tests** – Add tests for the resizing logic, edge cases (empty image, unsupported format), and for the interaction with the `ProductImagePut/Get/Remove` collaborators using mocks.  

---

**Overall**, the class provides the necessary plumbing to manage product images but would benefit from tighter resource handling, clearer error reporting, and removal of legacy commented code. Implementing the above recommendations will improve reliability, maintainability, and clarity for future contributors.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.cms.product;

import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;
import java.net.FileNameMap;
import java.net.URLConnection;
import java.util.List;
import javax.imageio.ImageIO;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.salesmanager.core.business.constants.Constants;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.utils.CoreConfiguration;
import com.salesmanager.core.business.utils.ProductImageCropUtils;
import com.salesmanager.core.business.utils.ProductImageSizeUtils;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;


public class ProductFileManagerImpl extends ProductFileManager {

  private static final Logger LOGGER = LoggerFactory.getLogger(ProductFileManagerImpl.class);


  private ProductImagePut uploadImage;
  private ProductImageGet getImage;
  private ProductImageRemove removeImage;

  private CoreConfiguration configuration;

  private final static String PRODUCT_IMAGE_HEIGHT_SIZE = "PRODUCT_IMAGE_HEIGHT_SIZE";
  private final static String PRODUCT_IMAGE_WIDTH_SIZE = "PRODUCT_IMAGE_WIDTH_SIZE";
  private final static String CROP_UPLOADED_IMAGES = "CROP_UPLOADED_IMAGES";


  public CoreConfiguration getConfiguration() {
    return configuration;
  }


  public void setConfiguration(CoreConfiguration configuration) {
    this.configuration = configuration;
  }


  public ProductImageRemove getRemoveImage() {
    return removeImage;
  }


  public void setRemoveImage(ProductImageRemove removeImage) {
    this.removeImage = removeImage;
  }


  public void addProductImage(ProductImage productImage, ImageContentFile contentImage)
      throws ServiceException {


    try {

      /** copy to input stream **/
      ByteArrayOutputStream baos = new ByteArrayOutputStream();
      // Fake code simulating the copy
      // You can generally do better with nio if you need...
      // And please, unlike me, do something about the Exceptions :D
      byte[] buffer = new byte[1024];
      int len;
      while ((len = contentImage.getFile().read(buffer)) > -1) {
        baos.write(buffer, 0, len);
      }
      baos.flush();

      // Open new InputStreams using the recorded bytes
      // Can be repeated as many times as you wish
      InputStream is1 = new ByteArrayInputStream(baos.toByteArray());
      InputStream is2 = new ByteArrayInputStream(baos.toByteArray());

      BufferedImage bufferedImage = ImageIO.read(is2);


      if (bufferedImage == null) {
        LOGGER.error("Cannot read image format for " + productImage.getProductImage());
        throw new Exception("Cannot read image format " + productImage.getProductImage());
      }

      // contentImage.setBufferedImage(bufferedImage);
      contentImage.setFile(is1);


      // upload original -- L
      contentImage.setFileContentType(FileContentType.PRODUCTLG);
      uploadImage.addProductImage(productImage, contentImage);

      /*
       * //default large InputContentImage largeContentImage = new
       * InputContentImage(ImageContentType.PRODUCT);
       * largeContentImage.setFile(contentImage.getFile());
       * largeContentImage.setDefaultImage(productImage.isDefaultImage());
       * largeContentImage.setImageName(new
       * StringBuilder().append("L-").append(productImage.getProductImage()).toString());
       *
       *
       * uploadImage.uploadProductImage(configuration, productImage, largeContentImage);
       */

      /*
       * //default small InputContentImage smallContentImage = new
       * InputContentImage(ImageContentType.PRODUCT);
       * smallContentImage.setFile(contentImage.getFile());
       * smallContentImage.setDefaultImage(productImage.isDefaultImage());
       * smallContentImage.setImageName(new
       * StringBuilder().append("S-").append(productImage.getProductImage()).toString());
       *
       * uploadImage.uploadProductImage(configuration, productImage, smallContentImage);
       */


      // get template properties file

      String slargeImageHeight = configuration.getProperty(PRODUCT_IMAGE_HEIGHT_SIZE);
      String slargeImageWidth = configuration.getProperty(PRODUCT_IMAGE_WIDTH_SIZE);

      // String ssmallImageHeight = configuration.getProperty("SMALL_IMAGE_HEIGHT_SIZE");
      // String ssmallImageWidth = configuration.getProperty("SMALL_IMAGE_WIDTH_SIZE");

      //Resizes
      if (!StringUtils.isBlank(slargeImageHeight) && !StringUtils.isBlank(slargeImageWidth)) { // &&
                                                                                               // !StringUtils.isBlank(ssmallImageHeight)
                                                                                               // &&
                                                                                               // !StringUtils.isBlank(ssmallImageWidth))
                                                                                               // {


        FileNameMap fileNameMap = URLConnection.getFileNameMap();

        String contentType = fileNameMap.getContentTypeFor(contentImage.getFileName());
        String extension = null;
        if (contentType != null) {
          extension = contentType.substring(contentType.indexOf('/') + 1, contentType.length());
        }

        if (extension == null) {
          extension = "jpeg";
        }


        int largeImageHeight = Integer.parseInt(slargeImageHeight);
        int largeImageWidth = Integer.parseInt(slargeImageWidth);

        if (largeImageHeight <= 0 || largeImageWidth <= 0) {
          String sizeMsg =
              "Image configuration set to an invalid value [PRODUCT_IMAGE_HEIGHT_SIZE] "
                  + largeImageHeight + " , [PRODUCT_IMAGE_WIDTH_SIZE] " + largeImageWidth;
          LOGGER.error(sizeMsg);
          throw new ServiceException(sizeMsg);
        }

        if (!StringUtils.isBlank(configuration.getProperty(CROP_UPLOADED_IMAGES))
            && configuration.getProperty(CROP_UPLOADED_IMAGES).equals(Constants.TRUE)) {
          // crop image
          ProductImageCropUtils utils =
              new ProductImageCropUtils(bufferedImage, largeImageWidth, largeImageHeight);
          if (utils.isCropeable()) {
            bufferedImage = utils.getCroppedImage();
          }
        }

        // do not keep a large image for now, just take care of the regular image and a small image

        // resize large
        // ByteArrayOutputStream output = new ByteArrayOutputStream();
        BufferedImage largeResizedImage;
        if(bufferedImage.getWidth() > largeImageWidth ||bufferedImage.getHeight() > largeImageHeight) {
            largeResizedImage = ProductImageSizeUtils.resizeWithRatio(bufferedImage, largeImageWidth, largeImageHeight);
        } else {
            largeResizedImage = bufferedImage;
        }


        File tempLarge =
            File.createTempFile(new StringBuilder().append(productImage.getProduct().getId())
                .append("tmpLarge").toString(), "." + extension);
        ImageIO.write(largeResizedImage, extension, tempLarge);

        try(FileInputStream isLarge = new FileInputStream(tempLarge)) {


        // IOUtils.copy(isLarge, output);


        ImageContentFile largeContentImage = new ImageContentFile();
        largeContentImage.setFileContentType(FileContentType.PRODUCT);
        largeContentImage.setFileName(productImage.getProductImage());
        largeContentImage.setFile(isLarge);


        // largeContentImage.setBufferedImage(bufferedImage);

        // largeContentImage.setFile(output);
        // largeContentImage.setDefaultImage(false);
        // largeContentImage.setImageName(new
        // StringBuilder().append("L-").append(productImage.getProductImage()).toString());


        uploadImage.addProductImage(productImage, largeContentImage);

        // output.flush();
        // output.close();

        tempLarge.delete();

        // now upload original



        /*
         * //resize small BufferedImage smallResizedImage = ProductImageSizeUtils.resize(cropped,
         * smallImageWidth, smallImageHeight); File tempSmall = File.createTempFile(new
         * StringBuilder().append(productImage.getProduct().getId()).append("tmpSmall").toString(),
         * "." + extension ); ImageIO.write(smallResizedImage, extension, tempSmall);
         *
         * //byte[] is = IOUtils.toByteArray(new FileInputStream(tempSmall));
         *
         * FileInputStream isSmall = new FileInputStream(tempSmall);
         *
         * output = new ByteArrayOutputStream(); IOUtils.copy(isSmall, output);
         *
         *
         * smallContentImage = new InputContentImage(ImageContentType.PRODUCT);
         * smallContentImage.setFile(output); smallContentImage.setDefaultImage(false);
         * smallContentImage.setImageName(new
         * StringBuilder().append("S-").append(productImage.getProductImage()).toString());
         *
         * uploadImage.uploadProductImage(configuration, productImage, smallContentImage);
         *
         * output.flush(); output.close();
         *
         * tempSmall.delete();
         */


    }
      } else {
        // small will be the same as the original
        contentImage.setFileContentType(FileContentType.PRODUCT);
        uploadImage.addProductImage(productImage, contentImage);
      }



    } catch (Exception e) {
      throw new ServiceException(e);
    } finally {
      try {
        productImage.getImage().close();
      } catch (Exception ignore) {
      }
    }

  }


  public OutputContentFile getProductImage(ProductImage productImage) throws ServiceException {
    // will return original
    return getImage.getProductImage(productImage);
  }


  @Override
  public List<OutputContentFile> getImages(final String merchantStoreCode,
      FileContentType imageContentType) throws ServiceException {
    // will return original
    return getImage.getImages(merchantStoreCode, FileContentType.PRODUCT);
  }

  @Override
  public List<OutputContentFile> getImages(Product product) throws ServiceException {
    return getImage.getImages(product);
  }



  @Override
  public void removeProductImage(ProductImage productImage) throws ServiceException {

    this.removeImage.removeProductImage(productImage);

    /*
     * ProductImage large = new ProductImage(); large.setProduct(productImage.getProduct());
     * large.setProductImage("L" + productImage.getProductImage());
     *
     * this.removeImage.removeProductImage(large);
     *
     * ProductImage small = new ProductImage(); small.setProduct(productImage.getProduct());
     * small.setProductImage("S" + productImage.getProductImage());
     *
     * this.removeImage.removeProductImage(small);
     */

  }


  @Override
  public void removeProductImages(Product product) throws ServiceException {

    this.removeImage.removeProductImages(product);

  }


  @Override
  public void removeImages(final String merchantStoreCode) throws ServiceException {

    this.removeImage.removeImages(merchantStoreCode);

  }


  public ProductImagePut getUploadImage() {
    return uploadImage;
  }


  public void setUploadImage(ProductImagePut uploadImage) {
    this.uploadImage = uploadImage;
  }



  public ProductImageGet getGetImage() {
    return getImage;
  }


  public void setGetImage(ProductImageGet getImage) {
    this.getImage = getImage;
  }


  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName) throws ServiceException {
    return getImage.getProductImage(merchantStoreCode, productCode, imageName);
  }



  @Override
  public OutputContentFile getProductImage(String merchantStoreCode, String productCode,
      String imageName, ProductImageSize size) throws ServiceException {
    return getImage.getProductImage(merchantStoreCode, productCode, imageName, size);
  }



}



```
