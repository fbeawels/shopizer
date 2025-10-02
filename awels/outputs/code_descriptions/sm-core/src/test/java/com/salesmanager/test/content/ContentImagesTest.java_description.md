# ContentImagesTest.java

## Review

## 1. Summary  
The `ContentImagesTest` class is a JUnit‑style integration test that exercises the `ContentService` layer of the SalesManager core by uploading a store logo image, persisting it via the CMS, verifying its retrieval, and finally deleting it.  
Key elements:

| Component | Role |
|-----------|------|
| `ContentService` | Service façade for managing binary content (logos, documents, etc.) |
| `InputContentFile` / `OutputContentFile` | DTOs representing a file to be uploaded or a file returned by the service |
| `FileContentType` | Enum that distinguishes content types (e.g. `LOGO`) |
| `MerchantStore` | Domain object representing a merchant's storefront |
| `merchantService` (inherited) | DAO/service for loading and persisting `MerchantStore` entities |
| `IOUtils` | Apache Commons helper for reading file bytes |

The test demonstrates the typical **Service‑DAO** interaction: data is read from the filesystem, wrapped in a DTO, passed to a service, persisted to a repository (likely a database or file store), and then retrieved back for verification.

---

## 2. Detailed Description  

### Flow of Execution  

1. **Retrieve the default store**  
   ```java
   MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
   ```

2. **Load an image from the local filesystem**  
   * The file is hard‑coded as `C:/doc/Hadoop.jpg`.  
   * It is validated for existence and readability, then read fully into a byte array via `IOUtils.toByteArray(new FileInputStream(file1))`.  
   * A `ByteArrayInputStream` is created from that byte array.

3. **Wrap the image in an `InputContentFile`**  
   * `cmsContentImage.setFileName(file1.getName());`  
   * `cmsContentImage.setFile(inputStream);`

4. **Persist the logo through `ContentService`**  
   ```java
   contentService.addLogo(store.getCode(), cmsContentImage);
   ```

5. **Update the `MerchantStore` entity with the new logo name**  
   ```java
   store.setStoreLogo(file1.getName());
   merchantService.update(store);
   ```

6. **Retrieve the store again** to ensure the persistence layer refreshed the entity.

7. **Fetch the stored logo**  
   ```java
   OutputContentFile image = contentService.getContentFile(store.getCode(), FileContentType.LOGO, logo);
   ```

8. **Write the retrieved file back to disk** (`C:/doc/logo-<filename>`).

9. **Clean up** by deleting the stored file from the CMS.  
   ```java
   contentService.removeFile(store.getCode(), FileContentType.LOGO, store.getStoreLogo());
   ```

### Dependencies & Assumptions  

* Relies on the Spring (or CDI) container for `@Inject` wiring of `ContentService`.  
* The test class extends `AbstractSalesManagerCoreTestCase`, which presumably boots the full application context.  
* Uses Apache Commons IO (`IOUtils`) for file handling.  
* Assumes the local filesystem is writable and that the `merchantService` bean is available.  

### Architectural Notes  

* **Service Layer**: `ContentService` encapsulates all persistence logic for binary data, exposing a clean API (`addLogo`, `getContentFile`, `removeFile`).  
* **DTO Pattern**: `InputContentFile` and `OutputContentFile` act as data carriers, decoupling the storage format from the business layer.  
* **Enum for Type Safety**: `FileContentType` ensures only supported content kinds are requested.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `createStoreLogo()` | Integration test for uploading, retrieving, and deleting a store logo. | None | None | Modifies `MerchantStore` entity, persists binary data, writes temporary files to disk. |

**Note**: No helper methods exist; all logic is inline.

---

## 4. Dependencies  

| Library | Version | Scope | Notes |
|---------|---------|-------|-------|
| Spring / CDI | N/A | Runtime | For `@Inject` injection. |
| JUnit 4 (`@Ignore`) | 4.x | Test | Provides test lifecycle. |
| Apache Commons IO (`IOUtils`) | 2.x | Runtime | Simplifies byte array handling. |
| SalesManager Core | N/A | Runtime | Contains `ContentService`, `MerchantService`, domain models. |

No explicit platform‑specific dependencies beyond the Windows path `C:/doc/...`, which limits portability.

---

## 5. Additional Notes & Recommendations  

### Strengths  
* Demonstrates end‑to‑end flow for binary content handling.  
* Clear separation of concerns via DTOs and service layer.  

### Weaknesses & Edge Cases  

1. **Resource Leaks** – `FileInputStream`, `ByteArrayOutputStream`, and `FileOutputStream` are never closed.  
   * **Fix**: Use try‑with‑resources or explicit `finally` blocks.  

2. **Hard‑coded Paths** – `C:/doc/...` is Windows‑only.  
   * **Fix**: Use `java.nio.file.Files.createTempFile()` or `System.getProperty("java.io.tmpdir")`.  

3. **Large File Handling** – Reading the entire file into memory can cause `OutOfMemoryError` for big logos.  
   * **Fix**: Stream the file directly to the service if supported.  

4. **Missing Assertions** – The test performs no verifications.  
   * **Fix**: Add `assertNotNull(image)` and compare byte arrays to confirm integrity.  

5. **Method Naming** – The method is named like a test but lacks `@Test`.  
   * **Fix**: Annotate with `@Test` and remove the class‑level `@Ignore`.  

6. **Exception Handling** – The method declares `throws` rather than using a test framework’s exception expectations.  
   * **Fix**: Let the test framework handle checked exceptions or catch and assert specific failure reasons.  

7. **Cleanup** – The test deletes the file via `ContentService`, but does not delete the temporary local copy.  
   * **Fix**: Delete the local file after the test completes.  

8. **Concurrency** – Not thread‑safe; if tests run in parallel, they might interfere with each other.  
   * **Fix**: Use unique temporary file names per test instance.  

### Future Enhancements  

* **Parameterized Tests** – Run the test against multiple file types (PNG, GIF) and sizes.  
* **Mocked File System** – Use a virtual file system (e.g., Jimfs) to avoid actual disk I/O.  
* **Integration with CI** – Ensure the test can run on non‑Windows agents by abstracting file paths.  
* **Performance Benchmarks** – Measure upload/download times for large images.  
* **Error Path Testing** – Validate behavior when the file is missing, corrupted, or the store code is invalid.  

---

### Quick Refactor Snippet  

```java
@Test
public void shouldPersistAndRetrieveStoreLogo() throws IOException {
    MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);

    Path tempImage = Files.createTempFile("logo", ".jpg");
    // copy a sample image into tempImage or generate a dummy byte array
    Files.write(tempImage, Files.readAllBytes(Paths.get("src/test/resources/sample-logo.jpg")));

    try (InputStream is = Files.newInputStream(tempImage)) {
        InputContentFile cmsContentImage = new InputContentFile();
        cmsContentImage.setFileName(tempImage.getFileName().toString());
        cmsContentImage.setFile(is);

        contentService.addLogo(store.getCode(), cmsContentImage);
        store.setStoreLogo(cmsContentImage.getFileName());
        merchantService.update(store);

        OutputContentFile image = contentService.getContentFile(
                store.getCode(), FileContentType.LOGO, store.getStoreLogo());

        assertNotNull(image);
        assertArrayEquals(Files.readAllBytes(tempImage), image.getFile().toByteArray());
    } finally {
        contentService.removeFile(store.getCode(), FileContentType.LOGO, store.getStoreLogo());
        Files.deleteIfExists(tempImage);
    }
}
```

This version closes streams, uses temporary files, performs an assertion, and cleans up after itself.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.content;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;

import javax.inject.Inject;

import org.apache.commons.io.IOUtils;
import org.junit.Ignore;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.content.ContentService;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;

/**
 * Test content with CMS store logo
 * 
 * @author Carl Samson
 *
 */
@Ignore
public class ContentImagesTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {

	@Inject
	private ContentService contentService;

	// @Test
	@Ignore
	public void createStoreLogo() throws ServiceException, FileNotFoundException, IOException {

		MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);

		final File file1 = new File("C:/doc/Hadoop.jpg");

		if (!file1.exists() || !file1.canRead()) {
			throw new ServiceException("Can't read" + file1.getAbsolutePath());
		}

		byte[] is = IOUtils.toByteArray(new FileInputStream(file1));
		ByteArrayInputStream inputStream = new ByteArrayInputStream(is);
		InputContentFile cmsContentImage = new InputContentFile();

		cmsContentImage.setFileName(file1.getName());
		cmsContentImage.setFile(inputStream);

		// logo as a content
		contentService.addLogo(store.getCode(), cmsContentImage);

		store.setStoreLogo(file1.getName());
		merchantService.update(store);

		// query the store
		store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);

		// get the logo
		String logo = store.getStoreLogo();

		OutputContentFile image = contentService.getContentFile(store.getCode(), FileContentType.LOGO, logo);

		// print image
		OutputStream outputStream = new FileOutputStream("C:/doc/logo-" + image.getFileName());

		ByteArrayOutputStream baos = image.getFile();
		baos.writeTo(outputStream);

		// remove image
		contentService.removeFile(store.getCode(), FileContentType.LOGO, store.getStoreLogo());

	}
	

}


```
