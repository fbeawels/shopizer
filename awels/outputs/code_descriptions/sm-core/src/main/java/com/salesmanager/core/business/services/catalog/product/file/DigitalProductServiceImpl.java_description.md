# DigitalProductServiceImpl.java

## Review

## 1. Summary

The **`DigitalProductServiceImpl`** class is a Spring‐managed service that handles CRUD operations for digital product files.  
It interacts with:

- **`DigitalProductRepository`** – persistence for `DigitalProduct` entities.  
- **`StaticContentFileManager`** – file system (or storage) abstraction for uploading/removing product files.  
- **`ProductService`** – to keep the parent `Product` entity in sync (e.g., toggling `productVirtual`).  

The service extends a generic `SalesManagerEntityServiceImpl` that already provides basic CRUD methods, and it implements the `DigitalProductService` interface.

---

## 2. Detailed Description

### Core Flow

| Step | Description |
|------|-------------|
| **Initialization** | Spring injects repository, file manager, and product service. The constructor calls `super(digitalProductRepository)` to wire the base CRUD layer. |
| **Adding a file** (`addProductFile`) | 1. Validates inputs (digital product, product, file, merchant store). <br>2. Persists the `DigitalProduct` via `saveOrUpdate`. <br>3. Delegates the physical file upload to `StaticContentFileManager.addFile`. <br>4. Marks the product as virtual and persists it. <br>5. Cleans up the input file stream in a `finally` block. |
| **Retrieving** (`getByProduct`) | Uses the repository to fetch the digital product by store and product IDs. |
| **Deleting** (`delete`) | 1. Validates the digital product and its linked product. <br>2. Reloads the entity from the database. <br>3. Calls the base `delete` method. <br>4. Removes the file from storage. <br>5. Marks the product as non‑virtual and updates it. |
| **Saving / Updating** (`saveOrUpdate`) | Determines whether to `save` or `create` (though the logic appears inverted). After persisting, it ensures the product is virtual and updates it. |

### Assumptions & Constraints

- **Product is always linked to a `MerchantStore`** – checked via `Assert`.
- **`StaticContentFileManager` expects a non‑null `path` (Optional)`** – current code passes `Optional.of(null)`, which will throw a `NullPointerException`.
- **`digitalProduct.getId()` may be `0` for new entities** – logic assumes `0` or `null` indicates creation.
- **File cleanup** is performed regardless of upload success; if upload fails, the file stream is still closed.

### Design Choices

- **Extension of a generic entity service** reduces boilerplate but blurs the boundary between domain logic and persistence logic.
- **Direct service injection** (ProductService) allows side‑effects (updating the parent product) to be encapsulated.
- **Use of Spring’s `Assert`** for defensive programming, though exceptions are caught and wrapped in `ServiceException`.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `addProductFile(Product, DigitalProduct, InputContentFile)` | Persist a `DigitalProduct` and store its file. | `Product`, `DigitalProduct`, `InputContentFile` | None (void) | - Updates `DigitalProduct` (id assigned).<br>- Stores file via `StaticContentFileManager`.<br>- Sets `productVirtual` to `true` and updates `Product`. |
| `getByProduct(MerchantStore, Product)` | Retrieve a digital product by store and product. | `MerchantStore`, `Product` | `DigitalProduct` (or `null`) | None |
| `delete(DigitalProduct)` | Remove a digital product and its file. | `DigitalProduct` | None | - Deletes entity from DB.<br>- Removes file via `StaticContentFileManager`.<br>- Sets `productVirtual` to `false` and updates `Product`. |
| `saveOrUpdate(DigitalProduct)` | Persist new or existing digital product. | `DigitalProduct` | None | - Calls `save` or `create` accordingly.<br>- Sets `productVirtual` to `true` and updates `Product`. |

### Utility / Helper Methods
- None beyond the inherited CRUD methods.

---

## 4. Dependencies

| Dependency | Type | Role |
|------------|------|------|
| `org.springframework.stereotype.Service` | Spring Framework | Marks the class as a service bean. |
| `org.springframework.util.Assert` | Spring Framework | Defensive checks. |
| `com.salesmanager.core.business.exception.ServiceException` | Application | Wraps lower‑level exceptions. |
| `com.salesmanager.core.business.modules.cms.content.StaticContentFileManager` | Application | Handles physical file storage. |
| `com.salesmanager.core.business.repositories.catalog.product.file.DigitalProductRepository` | Spring Data JPA | Repository interface for persistence. |
| `com.salesmanager.core.business.services.catalog.product.ProductService` | Application | Service for the parent `Product`. |
| `com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl` | Application | Generic CRUD base class. |
| `com.salesmanager.core.model.*` | Domain model | Entities and DTOs used in the service. |

All dependencies are internal to the `salesmanager` project, except for the Spring Framework.

---

## 5. Additional Notes

### Edge Cases & Potential Issues

1. **`Optional.of(path)` with `path == null`**  
   - Throws `NullPointerException`. Should use `Optional.ofNullable(path)` or `Optional.empty()`.

2. **File stream handling**  
   - The `finally` block blindly closes the stream if non‑null, but if the upload fails, the file may already be partially written. Consider transactionality or rollback logic.

3. **`saveOrUpdate` logic**  
   - `super.create(digitalProduct)` is called when `id` is non‑null, which may actually create a duplicate instead of updating. Likely a mistake; should call `super.update` or `super.save`.

4. **`delete` sequence**  
   - The entity is reloaded from the database (`getById`) before deletion, then `super.delete` is called. If the file removal fails after the DB delete, the product will be marked non‑virtual but the file may still exist, leading to orphaned files.

5. **Concurrency**  
   - No locking or transaction management is visible. Spring’s default behavior might open a transaction per method, but file operations are outside the transaction, which can leave the system in an inconsistent state.

6. **Hard‑coded `path` variable**  
   - The `path` variable is declared but never initialized. The intention is unclear; probably a placeholder for future path resolution logic.

### Suggested Enhancements

- **Transactional safety** – Wrap database and file operations in a single transaction where possible (e.g., use `@Transactional` and ensure the file manager supports rollback or compensating actions).
- **Refactor `saveOrUpdate`** – Replace the `create` call with an `update` for existing entities; consider delegating to `super.saveOrUpdate` if the base class provides it.
- **Error handling** – Provide more granular exception types (e.g., `FileStorageException`) instead of generic `ServiceException`.
- **Logging** – Add logging statements for key actions (file upload, deletion, errors) to aid debugging.
- **Unit tests** – Verify edge cases such as null file, missing merchant store, duplicate file uploads, and delete failures.
- **Path management** – Either remove the unused `path` variable or integrate it properly to allow custom storage paths.

---

**Overall**, the class encapsulates the domain logic for managing digital product files but contains a few bugs (null `Optional`, incorrect CRUD branch) and design concerns (transactional consistency, error handling). Addressing these points will improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.catalog.product.file;

import java.util.Optional;

import javax.inject.Inject;
import org.springframework.stereotype.Service;
import org.springframework.util.Assert;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.modules.cms.content.StaticContentFileManager;
import com.salesmanager.core.business.repositories.catalog.product.file.DigitalProductRepository;
import com.salesmanager.core.business.services.catalog.product.ProductService;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.file.DigitalProduct;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;

@Service("digitalProductService")
public class DigitalProductServiceImpl extends SalesManagerEntityServiceImpl<Long, DigitalProduct> 
	implements DigitalProductService {
	

	private DigitalProductRepository digitalProductRepository;
	
    @Inject
    StaticContentFileManager productDownloadsFileManager;
    
    @Inject
    ProductService productService;

	@Inject
	public DigitalProductServiceImpl(DigitalProductRepository digitalProductRepository) {
		super(digitalProductRepository);
		this.digitalProductRepository = digitalProductRepository;
	}
	
	@Override
	public void addProductFile(Product product, DigitalProduct digitalProduct, InputContentFile inputFile) throws ServiceException {
	
		Assert.notNull(digitalProduct,"DigitalProduct cannot be null");
		Assert.notNull(product,"Product cannot be null");
		digitalProduct.setProduct(product);

		try {
			
			Assert.notNull(inputFile.getFile(),"InputContentFile.file cannot be null");
			
			Assert.notNull(product.getMerchantStore(),"Product.merchantStore cannot be null");
			this.saveOrUpdate(digitalProduct);
			
		    String path = null;
		    
			
			productDownloadsFileManager.addFile(product.getMerchantStore().getCode(), Optional.of(path), inputFile);
			
			product.setProductVirtual(true);
			productService.update(product);
		
		} catch (Exception e) {
			throw new ServiceException(e);
		} finally {
			try {

				if(inputFile.getFile()!=null) {
					inputFile.getFile().close();
				}

			} catch(Exception ignore) {}
		}
		
		
	}
	
	@Override
	public DigitalProduct getByProduct(MerchantStore store, Product product) throws ServiceException {
		return digitalProductRepository.findByProduct(store.getId(), product.getId());
	}
	
	@Override
	public void delete(DigitalProduct digitalProduct) throws ServiceException {
		
		Assert.notNull(digitalProduct,"DigitalProduct cannot be null");
		Assert.notNull(digitalProduct.getProduct(),"DigitalProduct.product cannot be null");
		//refresh file
		digitalProduct = this.getById(digitalProduct.getId());
		super.delete(digitalProduct);
		
		String path = null;

		productDownloadsFileManager.removeFile(digitalProduct.getProduct().getMerchantStore().getCode(), FileContentType.PRODUCT, digitalProduct.getProductFileName(), Optional.of(path));
		digitalProduct.getProduct().setProductVirtual(false);
		productService.update(digitalProduct.getProduct());
	}
	
	
	@Override
	public void saveOrUpdate(DigitalProduct digitalProduct) throws ServiceException {
		
		Assert.notNull(digitalProduct,"DigitalProduct cannot be null");
		Assert.notNull(digitalProduct.getProduct(),"DigitalProduct.product cannot be null");
		if(digitalProduct.getId()==null || digitalProduct.getId() ==0) {
			super.save(digitalProduct);
		} else {
			super.create(digitalProduct);
		}
		
		digitalProduct.getProduct().setProductVirtual(true);
		productService.update(digitalProduct.getProduct());
		
		
	}
	

	

}



```
