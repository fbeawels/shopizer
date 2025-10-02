# ProductImageServiceImpl.java

## Review

## 1. Summary

**Purpose**  
`ProductImageServiceImpl` is the core service that manages product images in the SalesManager catalogue.  
It handles CRUD operations on `ProductImage` entities, orchestrates image storage through a `ProductFileManager`, and publishes domain events (save/delete) for other components to react.

**Key Components**

| Component | Responsibility |
|-----------|----------------|
| `ProductImageRepository` | Persistence of `ProductImage` entities (JPA/Hibernate). |
| `ProductFileManager` | Low‑level file operations (upload, retrieve, delete) on the storage backend. |
| `ApplicationEventPublisher` | Publishes `SaveProductImageEvent` and `DeleteProductImageEvent` for event‑driven side effects (e.g., caching, search indexing). |
| `ProductImageServiceImpl` | Orchestrates all above services, performs validation, error handling, and business‑logic. |

**Notable Design Patterns / Frameworks**

* **Service Layer** – Encapsulates business logic and coordinates other components.  
* **Repository Pattern** – Abstracts database access.  
* **Event Publishing** – Spring’s event system for decoupled side‑effects.  
* **Dependency Injection** – Spring / JSR‑330 (`@Inject`, `@Autowired`).  
* **Use of Optional** – Defensive API for potentially missing entities.

---

## 2. Detailed Description

### Flow of Execution

1. **Adding Images**  
   * `addProductImages` iterates over a list of `ProductImage` objects, converts each into an `ImageContentFile`, and delegates to `addProductImage`.  
   * `addProductImage` sets the owning product, validates the input stream, calls `productFileManager.addProductImage`, persists the entity, and publishes a `SaveProductImageEvent`.  
   * All exceptions are wrapped into `ServiceException`.  

2. **Removing an Image**  
   * `removeProductImage` delegates the file deletion to `productFileManager`, then deletes the JPA entity and publishes a `DeleteProductImageEvent`.  

3. **Retrieval**  
   * Image retrieval comes in two flavours:  
     * By `ProductImage` instance + size (`getProductImage(ProductImage, ProductImageSize)`) – constructs a new `ProductImage` with a size‑prefix in the file name and asks the file manager to fetch the content.  
     * By store/product code & filename (`getProductImage(String, String, String, ProductImageSize)`) – fully delegated to the file manager.  

4. **Other Operations**  
   * `addProductImageDescription` attaches a `ProductImageDescription` to an image and persists the change.  
   * `updateProductImage` validates input, re‑assigns the product, and saves the entity.  
   * `saveOrUpdate` simply forwards to the repository.

### Assumptions & Constraints

| Assumption | Rationale |
|------------|-----------|
| `productImage.getImageType() == 0` → file upload required | Likely an enum (or legacy int) indicating “actual image” vs. placeholder. |
| Image filenames are unique per product & size | Implicit in the `L-`/`S-` prefix logic. |
| File streams are closed by the caller or internally | `finally` block attempts to close the input stream. |
| Events must be fired after the entity is persisted | Manual event publishing compensates for missing AOP aspect. |

### Architecture

* **Layered** – Presentation → Service (this class) → Repository / File Manager.  
* **Event‑Driven** – Side effects (indexing, cache eviction) are decoupled via Spring events.  
* **Transactional** – Not explicitly annotated, but likely relies on `@Transactional` at the repository or service level in the parent class (`SalesManagerEntityServiceImpl`).  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getById(Long id)` | Retrieve a `ProductImage` by primary key. | `id` | `ProductImage` | None |
| `addProductImages(Product, List<ProductImage>)` | Bulk add product images (wrapping each in `ImageContentFile`). | `product`, `productImages` | `void` | Creates entities, stores files, publishes events |
| `addProductImage(Product, ProductImage, ImageContentFile)` | Single‑image addition (sets product, stores file, persists). | `product`, `productImage`, `inputImage` | `void` | File storage, DB persist, event |
| `saveOrUpdate(ProductImage)` | Persist or update an image entity. | `productImage` | `ProductImage` | DB operation |
| `addProductImageDescription(ProductImage, ProductImageDescription)` | Attach a description to an image. | `productImage`, `description` | `void` | Updates image, persists |
| `getProductImage(ProductImage, ProductImageSize)` | Return the image file for a specific size. | `productImage`, `size` | `OutputContentFile` | Reads file via file manager |
| `getProductImage(String, String, String, ProductImageSize)` | Retrieve image by store/product code & filename. | `storeCode`, `productCode`, `fileName`, `size` | `OutputContentFile` | Delegates to file manager |
| `getProductImages(Product)` | Fetch all images for a product. | `product` | `List<OutputContentFile>` | Delegates to file manager |
| `removeProductImage(ProductImage)` | Delete an image (file + DB record). | `productImage` | `void` | File deletion, DB delete, event |
| `getProductImage(Long, Long, MerchantStore)` | Optional retrieval by imageId + productId + store. | `imageId`, `productId`, `store` | `Optional<ProductImage>` | None |
| `updateProductImage(Product, ProductImage)` | Update an image’s product association. | `product`, `productImage` | `void` | DB update |

**Reusable/Utility Methods**

* `saveOrUpdate` – used by `addProductImage` and `updateProductImage`.  
* `addProductImageDescription` – can be reused wherever descriptions are added.  

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.stereotype.Service` | Spring | Marks this as a Spring bean. |
| `org.springframework.beans.factory.annotation.Autowired` | Spring DI | Injects `ApplicationEventPublisher`. |
| `javax.inject.Inject` | JSR‑330 DI | Injects repository & file manager. |
| `org.apache.commons.lang3.StringUtils` | Apache Commons | For blank‑string checks. |
| `org.jsoup.helper.Validate` | **Potential bug** | Likely intended to be `org.apache.commons.lang3.Validate`. |
| `org.springframework.util.Assert` | Spring | For null checks. |
| `ProductFileManager` | Application | Handles actual file storage. |
| `ProductImageRepository` | Spring Data JPA | CRUD for `ProductImage`. |
| `ApplicationEventPublisher` | Spring | Publishes domain events. |
| `OutputContentFile`, `ImageContentFile`, `FileContentType` | Application | Content wrappers. |

**Platform Specifics**

* Relies on Spring’s event system and JPA/Hibernate; assumes a relational database backing `ProductImage`.  
* Uses `InputStream` for file data; callers must manage stream lifecycles.  

---

## 5. Additional Notes & Recommendations

### Edge Cases / Potential Issues

1. **Incorrect `Validate` Import**  
   * The class imports `org.jsoup.helper.Validate`, which is unrelated to Commons Lang. It may compile but will not provide the expected validation semantics.  
   * **Fix**: Replace with `org.apache.commons.lang3.Validate` or use Spring’s `Assert`.

2. **Inconsistent DI Annotations**  
   * Mixing `@Inject` and `@Autowired` is fine, but consistency is preferred for readability.  
   * **Fix**: Stick to one style (preferably Spring’s `@Autowired` or JSR‑330 `@Inject`).

3. **Resource Leak in `addProductImage`**  
   * The `finally` block closes the stream only if it’s non‑null. However, if an exception occurs before the stream is created, `inputImage.getFile()` may be `null`. The code handles that, but it would be cleaner to use **try‑with‑resources** or delegate closing to the `ProductFileManager`.

4. **Deprecated Repository Method**  
   * `productImageRepository.findOne(id)` is deprecated in newer Spring Data JPA. Consider `findById(id)` returning `Optional`.  
   * **Fix**: Update the repository to expose `Optional<ProductImage> findById(Long id)` and adjust callers accordingly.

5. **Hard‑coded Image Size Prefixes**  
   * The logic that prefixes `L-` or `S-` to the file name is brittle. It assumes the storage layer uses this naming convention.  
   * **Fix**: Encapsulate size handling in `ProductFileManager` or use a dedicated naming strategy.

6. **Lack of Null/Empty Checks**  
   * Methods such as `removeProductImage` and `addProductImageDescription` assume non‑null arguments. While the service may be used internally, defensive checks (or `@NotNull` annotations) would make the API safer.

7. **Event Publishing Workarounds**  
   * The comments (“workaround for aspect”) imply that an AOP interceptor was expected to publish events. This manual publishing couples the service to event logic.  
   * **Fix**: If an aspect is desired, implement a proper `@AfterReturning` advice on the repository or service methods.

8. **Error Handling Granularity**  
   * The blanket `catch (Exception e)` can swallow important runtime exceptions (e.g., `NullPointerException`).  
   * **Fix**: Catch specific checked exceptions (`IOException`, `DataAccessException`) or rethrow runtime ones after logging.

### Suggested Enhancements

| Area | Improvement |
|------|-------------|
| **Event Strategy** | Use Spring’s `@TransactionalEventListener` or a dedicated interceptor to automatically fire events on persistence. |
| **Image Size Handling** | Create an enum `ProductImageSize` with helper methods for filename transformation, eliminating magic strings. |
| **Validation** | Centralize validation logic (e.g., via a `ProductImageValidator` component). |
| **Repository Naming** | Rename `productImageRepository.finById` to `findById` (typo). |
| **Logging** | Add structured logging (SLF4J) for successes/failures. |
| **Unit Tests** | Write tests for each method, especially edge cases (null inputs, missing files). |
| **Performance** | Batch inserts for bulk image uploads; use Spring’s `@Transactional` to reduce round‑trips. |
| **Security** | Validate that the authenticated user owns the merchant store before adding/removing images. |

### Overall Assessment

`ProductImageServiceImpl` provides a solid, service‑layer abstraction over the underlying persistence and file storage concerns. The use of Spring events and a dedicated file manager decouples concerns nicely. However, the code has a few technical debt items (deprecated methods, inconsistent DI, potential validation bug) that should be addressed to improve maintainability, readability, and robustness. Once those are fixed and the suggestions above are considered, the service will be a clean, well‑tested component suitable for production use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.product.image;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import javax.inject.Inject;

import org.apache.commons.lang3.StringUtils;
import org.jsoup.helper.Validate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.util.Assert;

import com.salesmanager.core.business.configuration.events.products.DeleteProductImageEvent;
import com.salesmanager.core.business.configuration.events.products.SaveProductImageEvent;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.product.ProductFileManager;
import com.salesmanager.core.business.repositories.catalog.product.image.ProductImageRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.ProductImageSize;
import com.salesmanager.core.model.catalog.product.image.ProductImage;
import com.salesmanager.core.model.catalog.product.image.ProductImageDescription;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.ImageContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;

@Service("productImage")
public class ProductImageServiceImpl extends SalesManagerEntityServiceImpl<Long, ProductImage>
		implements ProductImageService {

	private ProductImageRepository productImageRepository;

	@Inject
	public ProductImageServiceImpl(ProductImageRepository productImageRepository) {
		super(productImageRepository);
		this.productImageRepository = productImageRepository;
	}

	@Inject
	private ProductFileManager productFileManager;
	
	@Autowired
	private ApplicationEventPublisher eventPublisher;

	public ProductImage getById(Long id) {

		return productImageRepository.findOne(id);
	}

	@Override
	public void addProductImages(Product product, List<ProductImage> productImages) throws ServiceException {

		try {
			for (ProductImage productImage : productImages) {

				Assert.notNull(productImage.getImage());

				InputStream inputStream = productImage.getImage();
				ImageContentFile cmsContentImage = new ImageContentFile();
				cmsContentImage.setFileName(productImage.getProductImage());
				cmsContentImage.setFile(inputStream);
				cmsContentImage.setFileContentType(FileContentType.PRODUCT);

				addProductImage(product, productImage, cmsContentImage);
			}

		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	@Override
	public void addProductImage(Product product, ProductImage productImage, ImageContentFile inputImage)
			throws ServiceException {

		productImage.setProduct(product);

		try {
			if (productImage.getImageType() == 0) {
				Assert.notNull(inputImage.getFile(), "ImageContentFile.file cannot be null");
				productFileManager.addProductImage(productImage, inputImage);
			}

			// insert ProductImage
			ProductImage img = saveOrUpdate(productImage);
			//manual workaround since aspect is not working
			eventPublisher.publishEvent(new SaveProductImageEvent(eventPublisher, img, product));

		} catch (Exception e) {
			throw new ServiceException(e);
		} finally {
			try {

				if (inputImage.getFile() != null) {
					inputImage.getFile().close();
				}

			} catch (Exception ignore) {

			}
		}

	}

	@Override
	public ProductImage saveOrUpdate(ProductImage productImage) throws ServiceException {

		return productImageRepository.save(productImage);

	}

	public void addProductImageDescription(ProductImage productImage, ProductImageDescription description)
			throws ServiceException {

		if (productImage.getDescriptions() == null) {
			productImage.setDescriptions(new ArrayList<ProductImageDescription>());
		}

		productImage.getDescriptions().add(description);
		description.setProductImage(productImage);
		update(productImage);

	}

	// TODO get default product image

	@Override
	public OutputContentFile getProductImage(ProductImage productImage, ProductImageSize size) throws ServiceException {

		ProductImage pi = new ProductImage();
		String imageName = productImage.getProductImage();
		if (size == ProductImageSize.LARGE) {
			imageName = "L-" + imageName;
		}

		if (size == ProductImageSize.SMALL) {
			imageName = "S-" + imageName;
		}

		pi.setProductImage(imageName);
		pi.setProduct(productImage.getProduct());

		return productFileManager.getProductImage(pi);

	}

	@Override
	public OutputContentFile getProductImage(final String storeCode, final String productCode, final String fileName,
			final ProductImageSize size) throws ServiceException {
		return productFileManager.getProductImage(storeCode, productCode, fileName, size);

	}

	@Override
	public List<OutputContentFile> getProductImages(Product product) throws ServiceException {
		return productFileManager.getImages(product);
	}

	@Override
	public void removeProductImage(ProductImage productImage) throws ServiceException {

		if (!StringUtils.isBlank(productImage.getProductImage())) {
			productFileManager.removeProductImage(productImage);// managed internally
		}
		ProductImage p = getById(productImage.getId());
		
		Product product = p.getProduct();
		
		
		delete(p);
		/**
		 * workaround for aspect
		 */
		eventPublisher.publishEvent(new DeleteProductImageEvent(eventPublisher, p, product));


	}

	@Override
	public Optional<ProductImage> getProductImage(Long imageId, Long productId, MerchantStore store) {

		Optional<ProductImage> image = Optional.empty();

		ProductImage img = productImageRepository.finById(imageId, productId, store.getCode());
		if (img != null) {
			image = Optional.of(img);
		}

		return image;
	}

	@Override
	public void updateProductImage(Product product, ProductImage productImage) {
		Validate.notNull(product, "Product cannot be null");
		Validate.notNull(productImage, "ProductImage cannot be null");
		productImage.setProduct(product);
		productImageRepository.save(productImage);

	}
}



```
