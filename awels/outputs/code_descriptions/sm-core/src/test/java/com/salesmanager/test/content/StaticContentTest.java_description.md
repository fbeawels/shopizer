# StaticContentTest.java

## Review

## 1. Summary  
The **`StaticContentTest`** class is a JUnit test skeleton that demonstrates how to interact with the `ContentService` for uploading, retrieving, and deleting static image files (e.g., PNG/JPG) in the SalesManager core framework. It performs the following high‑level steps:

1. Loads an image from a hard‑coded filesystem path.  
2. Wraps the image bytes in an `InputContentFile` DTO and sends it to `ContentService.addContentFile`.  
3. Retrieves the same file via `ContentService.getContentFile` and writes it back to disk.  
4. Finally, removes the file with `ContentService.removeFile`.  

Key components  
- `ContentService` – the core service handling file persistence.  
- `InputContentFile` / `OutputContentFile` – DTOs that carry file metadata and streams.  
- `FileContentType` – enum used to indicate the file type (image, static, etc.).  
- `MerchantStore` – represents the store context for the operation.  

Design patterns / libraries  
- Dependency injection via `@Inject` (likely CDI or Spring).  
- Apache Commons IO (`IOUtils`) for convenient stream conversion.  
- JUnit 4 (`@Test`, `@Ignore`) for test orchestration.  

## 2. Detailed Description  
The test case runs inside the SalesManager test harness (`AbstractSalesManagerCoreTestCase`), which probably bootstraps the Spring context and provides services such as `merchantService` and `contentService`.  

**Execution flow**  
1. **Setup** – `merchantService.getByCode(MerchantStore.DEFAULT_STORE)` fetches the default store.  
2. **Reading the source file** – The file path is hard‑coded (`IMAGE_FILE`). The test verifies readability, then reads the entire file into a byte array via `IOUtils.toByteArray`.  
3. **Uploading** – An `InputContentFile` is populated with the file name, input stream, and type `IMAGE`. It is passed to `contentService.addContentFile`.  
4. **Downloading** – `contentService.getContentFile` returns an `OutputContentFile`. The code writes the returned `ByteArrayOutputStream` back to disk under `OUTPUT_FOLDER`.  
5. **Cleanup** – The file is removed from the store using `contentService.removeFile`.  

The test has no assertions; it merely exercises the API and would fail if an exception is thrown. The `@Ignore` annotation suppresses execution, so it functions more as a usage example than an automated test.

### Assumptions & Constraints  
- The image file exists at the specified path and is readable.  
- The store code `MerchantStore.DEFAULT_STORE` is present in the test database.  
- The underlying `ContentService` can handle the provided `InputContentFile`/`OutputContentFile` DTOs.  
- The test runs on a filesystem with write permission to `OUTPUT_FOLDER`.  

### Design Choices  
- Using a byte array to hold the file is simple but not memory‑efficient for large files.  
- The DTO approach keeps the service contract clear but may hide stream handling details.  
- The test bypasses assertions; its purpose is more illustrative than verification‑driven.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `createImage()` | Test entry point. Demonstrates add‑get‑remove lifecycle for an image. | None (uses class‑level constants) | None | Interacts with `ContentService`; writes to disk; may throw `ServiceException`, `FileNotFoundException`, `IOException`. |

The method contains several inline steps rather than helper methods. Potential reusable pieces could be extracted:

- **`readFileAsByteArray(File)`** – encapsulates reading file content.  
- **`writeStreamToFile(ByteArrayOutputStream, String)`** – abstracts writing to disk.  

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.business.services.content.ContentService` | Third‑party (SalesManager core) | Core service for content CRUD. |
| `com.salesmanager.core.model.content.*` | Third‑party | DTOs and enums used by `ContentService`. |
| `com.salesmanager.core.business.exception.ServiceException` | Third‑party | Custom exception for service errors. |
| `com.salesmanager.core.model.merchant.MerchantStore` | Third‑party | Entity representing a merchant store. |
| `org.apache.commons.io.IOUtils` | Third‑party | Utility class for stream handling. |
| `org.junit.Test`, `org.junit.Ignore` | Third‑party | JUnit 4 annotations. |
| `javax.inject.Inject` | Standard (JSR‑330) | CDI/Spring injection. |

No platform‑specific APIs are used beyond standard Java I/O.  

## 5. Additional Notes  

### Edge Cases / Potential Pitfalls  
- **Memory consumption**: `IOUtils.toByteArray` loads the entire file into RAM, which may fail for large media.  
- **Resource leaks**: `FileInputStream`, `FileOutputStream`, and `ByteArrayInputStream` are never closed. Although the JVM will clean them up eventually, they should be closed explicitly (preferably via try‑with‑resources).  
- **Hard‑coded paths**: Makes the test fragile. Switching environments (CI, other OSes) will require editing the source.  
- **No assertions**: The test will pass as long as no exception is thrown, which may hide functional defects.  
- **Output folder permission**: If `OUTPUT_FOLDER` does not exist or lacks write permission, the test will fail.  
- **Service assumptions**: If `contentService.addContentFile` stores data asynchronously, subsequent `getContentFile` may return stale data.

### Suggested Enhancements  
1. **Resource Management**  
   ```java
   try (FileInputStream fis = new FileInputStream(file1);
        ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
       IOUtils.copy(fis, baos);
       // use baos.toByteArray()
   }
   ```
2. **Assertions**  
   - Verify that the retrieved file name matches the original.  
   - Compare checksums of original and retrieved files.  
3. **Parameterize Test**  
   Use JUnit `Parameterized` or `@RunWith(Parameterized.class)` to test multiple file types/paths without hard‑coding.  
4. **Temporary Directories**  
   Use `java.nio.file.Files#createTempDirectory` to avoid cluttering the user's workspace.  
5. **Memory‑efficient Streaming**  
   Pass a `FileInputStream` directly to `InputContentFile` if the service supports streaming, rather than loading into a byte array.  
6. **Logging**  
   Add informational logs to aid debugging when the test is enabled.  

### Overall Impression  
The class serves as a quick manual example of using `ContentService` for image files. It is functional but would benefit from modern Java best practices (try‑with‑resources, assertions, parameterization) and from decoupling file paths from source code. Once refined, it can evolve into a robust automated test for content handling within the SalesManager platform.

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
import org.junit.Test;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.content.ContentService;
import com.salesmanager.core.model.content.FileContentType;
import com.salesmanager.core.model.content.InputContentFile;
import com.salesmanager.core.model.content.OutputContentFile;
import com.salesmanager.core.model.merchant.MerchantStore;



/**
 * Test 
 * 
 * - static content files (.js, .pdf etc)
 * - static content images (jpg, gig ...)
 * @author Carl Samson
 *
 */
@Ignore
public class StaticContentTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	

	@Inject
	private ContentService contentService;
	
	/**
	 * Change this path to an existing image path
	 */
	private final static String IMAGE_FILE = "/Users/carlsamson/Documents/Database.png";
	
	private final static String OUTPUT_FOLDER = "/Users/carlsamson/Documents/test/";
	
	
    @Test
    public void createImage()
        throws ServiceException, FileNotFoundException, IOException
    {

        MerchantStore store = merchantService.getByCode( MerchantStore.DEFAULT_STORE );
        final File file1 = new File( IMAGE_FILE);

        if ( !file1.exists() || !file1.canRead() )
        {
            throw new ServiceException( "Can't read" + file1.getAbsolutePath() );
        }

        final byte[] is = IOUtils.toByteArray( new FileInputStream( file1 ) );
        final ByteArrayInputStream inputStream = new ByteArrayInputStream( is );
        final InputContentFile cmsContentImage = new InputContentFile();
        cmsContentImage.setFileName( file1.getName() );
        cmsContentImage.setFile( inputStream );
        cmsContentImage.setFileContentType(FileContentType.IMAGE);
        
        //Add image
        contentService.addContentFile(store.getCode(), cmsContentImage);

    
        //get image
		OutputContentFile image = contentService.getContentFile(store.getCode(), FileContentType.IMAGE, file1.getName());

        //print image
   	 	OutputStream outputStream = new FileOutputStream (OUTPUT_FOLDER + image.getFileName()); 

   	 	ByteArrayOutputStream baos =  image.getFile();
   	 	baos.writeTo(outputStream);
		
		
		//remove image
   	 	contentService.removeFile(store.getCode(), FileContentType.IMAGE, file1.getName());
		


    }
	

}


```
