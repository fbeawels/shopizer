# ImagesController.java

## Review

## 1. Summary

`ImagesController` is a **Spring MVC** controller that exposes a set of HTTP endpoints for retrieving images and other binary files stored on the application server.  
The controller:

1. Loads a default “not‑found” PNG image at startup (used when a requested file cannot be located).  
2. Delegates the actual image retrieval to two services:
   * `ContentService` – for generic content such as logos, content images, or property files.  
   * `ProductImageService` – for product‑specific images, with support for size variants (`SMALL`, `LARGE`).  
3. Provides several URL patterns that map to overloaded `printImage` methods, each handling a different combination of path variables and query parameters.  
4. Returns raw image bytes (`byte[]`) directly as the HTTP response body.

The controller uses only standard Spring MVC annotations (`@Controller`, `@RequestMapping`, `@PathVariable`, `@ResponseBody`) and a handful of third‑party utilities (`Apache Commons Lang`, `ResourceUtils`).

---

## 2. Detailed Description

### 2.1 Initialization
- `@PostConstruct init()` loads `classpath:static/not-found.png` into a byte array (`tempImage`).  
- If the file is missing or unreadable, an error is logged and `tempImage` remains `null`.  
- The image is loaded once, shared across all requests.

### 2.2 URL Patterns & Methods
| Path | Parameters | Method | Description |
|------|------------|--------|-------------|
| `/static/files/{storeCode}/{imageType}/{imageName}.{extension}` | `storeCode`, `imageType`, `imageName`, `extension` | `printImage()` | Retrieves generic content (logo, content image, property) via `ContentService`. |
| `/static/{storeCode}/{imageType}/{productCode}/{imageName}.{extension}` | `storeCode`, `productCode`, `imageType`, `imageName`, `extension` | `printImage()` (Deprecated) | Legacy product image endpoint. |
| `/static/products/{storeCode}/{productCode}/{imageSize}/{imageName}.{extension}` | `storeCode`, `productCode`, `imageSize`, `imageName`, `extension` | `printImage()` | Product image with size encoded in the path (`SMALL`/`LARGE`). |
| `/static/products/{storeCode}/{productCode}/{imageName}.{extension}` | `storeCode`, `productCode`, `imageName`, `extension` | `printImage()` | Product image with optional `size` query param (`small`/`large`). |

All methods share the same core logic:
1. Resolve the requested image size or type.  
2. Call the appropriate service (`ContentService` or `ProductImageService`).  
3. If a file is returned, convert it to a byte array and send it back; otherwise return `tempImage`.

### 2.3 Flow of Execution
1. **HTTP Request** → matched to a `@RequestMapping`.  
2. **Path variables** extracted by Spring.  
3. **Service call** performed (may throw `ServiceException`).  
4. **Image returned** as raw bytes; if missing, the default image is sent.  
5. **No explicit cleanup** – the byte arrays are returned and GC handles them.

### 2.4 Assumptions & Constraints
- `tempImage` must be present in the classpath; otherwise null images are served.  
- `imageType` values are compared with `FileContentType` enum names; no case‑insensitivity handling.  
- No validation of `extension` – the service expects a valid filename.  
- The controller assumes all image data fits comfortably into memory (`byte[]`).  
- No caching headers or content‑type negotiation beyond the `produces` attribute.

### 2.5 Architecture & Design Choices
- **Thin controller** – logic is minimal, delegating to services.  
- **Method overloading** – same name (`printImage`) reused for clarity but can be confusing.  
- **Hard‑coded default image** – convenient but not flexible (cannot be overridden per store).  
- **Limited HTTP semantics** – no `ResponseEntity`, so content‑type and caching headers are not controllable.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `init()` | Loads default image at startup. | None | None | Sets `tempImage` or logs error |
| `printImage(storeCode, imageType, imageName, extension)` | Retrieve generic content. | `storeCode`, `imageType`, `imageName`, `extension` | `byte[]` image data or default image | None |
| `printImage(storeCode, productCode, imageType, imageName, extension)` *(Deprecated)* | Legacy product image retrieval. | `storeCode`, `productCode`, `imageType`, `imageName`, `extension` | `byte[]` | Logs errors |
| `printImage(storeCode, productCode, imageSize, imageName, extension, request)` | Product image with size in path. | `storeCode`, `productCode`, `imageSize`, `imageName`, `extension`, `HttpServletRequest` | `byte[]` | Logs errors |
| `printImage(storeCode, productCode, imageName, extension, request)` | Product image with size as query param. | `storeCode`, `productCode`, `imageName`, `extension`, `HttpServletRequest` | `byte[]` | Logs errors |

**Reusable utilities**  
- `new StringBuilder().append(imageName).append(".").append(extension).toString()` – could be extracted into a helper method.  
- `StringUtils.isNotBlank()` – used only in the last method for parameter validation.

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `spring-webmvc` | Core | Provides `@Controller`, `@RequestMapping`, etc. |
| `spring-context` | Core | Needed for `ResourceUtils`. |
| `commons-lang3` | Third‑party | Provides `StringUtils`. |
| `slf4j-api` & `slf4j-log4j12` (or similar) | Logging | For `Logger`. |
| Custom services | Application | `ContentService`, `ProductImageService`. |
| Custom model classes | Application | `OutputContentFile`, `FileContentType`, `ProductImageSize`. |

No database or external APIs are called directly from this controller; it relies entirely on injected services.

---

## 5. Additional Notes & Recommendations

### 5.1 Edge Cases & Potential Bugs
1. **Null `tempImage`** – if `not-found.png` is missing, the controller will return `null`, causing a `NullPointerException` when the framework tries to write the response.  
2. **Content‑Type Handling** – the controller does not set the `Content-Type` header when returning a byte array, except for the two methods that specify `produces`. For the generic content endpoint this could result in browsers guessing the type incorrectly.  
3. **Duplicate Method Names** – while Java allows overloading, it can make debugging harder (stack traces will show `printImage` without context).  
4. **Hard‑coded Size Mapping** – the logic that converts `imageSize` or `size` query param to `ProductImageSize` is repeated in two methods.  
5. **Deprecated Endpoint** – the legacy product image mapping is still alive; if removed, any existing clients would break.  

### 5.2 Performance & Scalability
- The controller loads the entire file into memory (`byte[]`) for every request. For large images or high traffic, this could lead to high memory pressure.  
- No caching headers (`Cache-Control`, `ETag`) are sent; browsers will re‑fetch images on every page load.

### 5.3 Suggested Enhancements
| Area | Suggestion |
|------|------------|
| **Response handling** | Return `ResponseEntity<byte[]>` to set `Content-Type` and caching headers explicitly. |
| **Error handling** | Return a proper HTTP status (e.g., 404) when image is missing instead of always returning the default image. |
| **Configuration** | Make the path to the default image configurable (e.g., via `@Value("${images.default}")`). |
| **DRY** | Extract common logic (size resolution, service call) into private helper methods. |
| **Logging** | Include request details (store code, image name) when logging errors. |
| **Unit tests** | Add tests for each mapping, especially verifying behavior when images are missing. |
| **Documentation** | Update Javadoc to reflect current endpoints and deprecation notes. |
| **Security** | Validate `storeCode`, `productCode`, and `imageName` to avoid path traversal or other injection attacks. |
| **Caching** | Add `@Cacheable` on service calls if images are static. |

### 5.4 Future Extensions
- **Dynamic image processing** – resizing on the fly based on query parameters (`width`, `height`).  
- **Support for other media types** – audio/video.  
- **User‑specific content** – e.g., user avatars with access control.  
- **Versioning** – append a version hash to URLs to force cache invalidation.  

---

**Overall**: The controller is functional and straightforward but can benefit from minor refactoring and more robust HTTP handling to improve reliability, maintainability, and performance.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.controller;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;

import javax.annotation.PostConstruct;
import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.util.ResourceUtils;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.catalog.product.image.ProductImageService;
import com.salesmanager.core.business.services.content.ContentService;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.OutputContentFile;

/**
 * When handling images and files from the application server
 * @author c.samson
 *
 */
@Controller
public class ImagesController {
	
	private static final Logger LOGGER = LoggerFactory.getLogger(ImagesController.class);
	

	
	@Inject
	private ContentService contentService;
	
	@Inject
	private ProductImageService productImageService;
	
	private byte[] tempImage = null;
	
	@PostConstruct
	public void init() {
		try {
			File file = ResourceUtils.getFile("classpath:static/not-found.png");
			if(file != null) {
				byte[] bFile = Files.readAllBytes(file.toPath());
				this.tempImage = bFile;
			}

			
		} catch (Exception e) {
			LOGGER.error("Can't load temporary default image", e);
		}
	}
	
	/**
	 * Logo, content image
	 * @param storeId
	 * @param imageType (LOGO, CONTENT, IMAGE)
	 * @param imageName
	 * @return
	 * @throws IOException
	 * @throws ServiceException 
	 */
	@RequestMapping("/static/files/{storeCode}/{imageType}/{imageName}.{extension}")
	public @ResponseBody byte[] printImage(@PathVariable final String storeCode, @PathVariable final String imageType, @PathVariable final String imageName, @PathVariable final String extension) throws IOException, ServiceException {

		// example -> /static/files/DEFAULT/CONTENT/myImage.png
		
		FileContentType imgType = null;
		
		if(FileContentType.LOGO.name().equals(imageType)) {
			imgType = FileContentType.LOGO;
		}
		
		if(FileContentType.IMAGE.name().equals(imageType)) {
			imgType = FileContentType.IMAGE;
		}
		
		if(FileContentType.PROPERTY.name().equals(imageType)) {
			imgType = FileContentType.PROPERTY;
		}
		
		OutputContentFile image =contentService.getContentFile(storeCode, imgType, new StringBuilder().append(imageName).append(".").append(extension).toString());
		
		
		if(image!=null) {
			return image.getFile().toByteArray();
		} else {
			return tempImage;
		}

	}
	

	/**
	 * For product images
	 * @Deprecated
	 * @param storeCode
	 * @param productCode
	 * @param imageType
	 * @param imageName
	 * @param extension
	 * @return
	 * @throws IOException
	 */
	@RequestMapping("/static/{storeCode}/{imageType}/{productCode}/{imageName}.{extension}")
	public @ResponseBody byte[] printImage(@PathVariable final String storeCode, @PathVariable final String productCode, @PathVariable final String imageType, @PathVariable final String imageName, @PathVariable final String extension) throws IOException {

		// product image
		// example small product image -> /static/DEFAULT/products/TB12345/product1.jpg
		
		// example large product image -> /static/DEFAULT/products/TB12345/product1.jpg

		
		/**
		 * List of possible imageType
		 * 
		 */
		

		ProductImageSize size = ProductImageSize.SMALL;
		
		if(imageType.equals(FileContentType.PRODUCTLG.name())) {
			size = ProductImageSize.LARGE;
		} 
		

		
		OutputContentFile image = null;
		try {
			image = productImageService.getProductImage(storeCode, productCode, new StringBuilder().append(imageName).append(".").append(extension).toString(), size);
		} catch (ServiceException e) {
			LOGGER.error("Cannot retrieve image " + imageName, e);
		}
		if(image!=null) {
			return image.getFile().toByteArray();
		} else {
			//empty image placeholder
			return tempImage;
		}

	}
	
	/**
	 * Exclusive method for dealing with product images
	 * @param storeCode
	 * @param productCode
	 * @param imageName
	 * @param extension
	 * @param request
	 * @return
	 * @throws IOException
	 */
	@RequestMapping(value="/static/products/{storeCode}/{productCode}/{imageSize}/{imageName}.{extension}",
			produces = {"image/gif", "image/jpg", "image/png", "application/octet-stream"})
	public @ResponseBody byte[] printImage(@PathVariable final String storeCode, @PathVariable final String productCode, @PathVariable final String imageSize, @PathVariable final String imageName, @PathVariable final String extension, HttpServletRequest request) throws IOException {

		// product image small
		// example small product image -> /static/products/DEFAULT/TB12345/SMALL/product1.jpg
		
		// example large product image -> /static/products/DEFAULT/TB12345/LARGE/product1.jpg


		/**
		 * List of possible imageType
		 * 
		 */
		
		
		ProductImageSize size = ProductImageSize.SMALL;
		
		if(FileContentType.PRODUCTLG.name().equals(imageSize)) {
			size = ProductImageSize.LARGE;
		} 
		
	

		
		OutputContentFile image = null;
		try {
			image = productImageService.getProductImage(storeCode, productCode, new StringBuilder().append(imageName).append(".").append(extension).toString(), size);
		} catch (ServiceException e) {
			LOGGER.error("Cannot retrieve image " + imageName, e);
		}
		if(image!=null) {
			return image.getFile().toByteArray();
		} else {
			//empty image placeholder
			return tempImage;
		}

	}
	
	/**
	 * Exclusive method for dealing with product images
	 * @param storeCode
	 * @param productCode
	 * @param imageName
	 * @param extension
	 * @param request
	 * @return
	 * @throws IOException
	 */
	@RequestMapping(value="/static/products/{storeCode}/{productCode}/{imageName}.{extension}",
	produces = {"image/gif", "image/jpg", "image/png", "application/octet-stream"})
	public @ResponseBody byte[] printImage(@PathVariable final String storeCode, @PathVariable final String productCode, @PathVariable final String imageName, @PathVariable final String extension, HttpServletRequest request) throws IOException {

		// product image
		// example small product image -> /static/products/DEFAULT/TB12345/product1.jpg?size=small
		
		// example large product image -> /static/products/DEFAULT/TB12345/product1.jpg
		// or
		//example large product image -> /static/products/DEFAULT/TB12345/product1.jpg?size=large
		

		/**
		 * List of possible imageType
		 * 
		 */
		

		ProductImageSize size = ProductImageSize.LARGE;
		
				
		if(StringUtils.isNotBlank(request.getParameter("size"))) {
			String requestSize = request.getParameter("size");
			if(requestSize.equals(ProductImageSize.SMALL.name())) {
				size = ProductImageSize.SMALL;
			} 
		}
		

		
		OutputContentFile image = null;
		try {
			image = productImageService.getProductImage(storeCode, productCode, new StringBuilder().append(imageName).append(".").append(extension).toString(), size);
		} catch (ServiceException e) {
			LOGGER.error("Cannot retrieve image " + imageName, e);
		}
		if(image!=null) {
			return image.getFile().toByteArray();
		} else {
			//empty image placeholder
			return tempImage;
		}

	}

}



```
