# MavenWrapperDownloader.java

## Review

## 1. Summary

The **`MavenWrapperDownloader`** is a tiny Java utility that downloads the Maven Wrapper JAR (`maven-wrapper.jar`) to a given project directory.  
It:

1. Accepts the base directory as a command‑line argument.  
2. Looks for `.mvn/wrapper/maven-wrapper.properties` and reads a `wrapperUrl` property if present.  
3. Falls back to a hard‑coded default URL (`https://repo.maven.apache.org/maven2/io/takari/maven-wrapper/0.4.2/maven-wrapper-0.4.2.jar`).  
4. Downloads the file to `.mvn/wrapper/maven-wrapper.jar`, creating intermediate directories as needed.  
5. Uses a `ReadableByteChannel` backed by the URL stream to copy the file in one call.

The code is deliberately minimal and relies only on the JDK (no external libraries).

---

## 2. Detailed Description

### High‑level Flow

| Step | Action | Notes |
|------|--------|-------|
| **1** | `main` reads the first argument → base directory. | No argument validation → NPE if omitted. |
| **2** | Checks for a `maven-wrapper.properties` file in the base directory. | Reads `wrapperUrl` if present; otherwise keeps the hard‑coded URL. |
| **3** | Builds the absolute path for `.mvn/wrapper/maven-wrapper.jar`. | Ensures parent directories exist (`mkdirs()`). |
| **4** | Calls `downloadFileFromURL(url, outputFile)`. | Uses `URL.openStream()` → `ReadableByteChannel` → `FileOutputStream`. |
| **5** | On success, prints “Done” and exits with status 0; on failure, prints stack trace and exits with status 1. | Uses `System.exit` – terminates the JVM directly. |

### Key Design Choices

| Choice | Rationale / Impact |
|--------|--------------------|
| **Hard‑coded URL/version** | Simplifies the tool but makes it brittle if the wrapper version changes. |
| **Single method for download** | Keeps the example concise but lacks HTTP error handling and progress reporting. |
| **`System.out` for logging** | Acceptable for a quick utility but not ideal for production use. |
| **`System.exit`** | Guarantees a clean exit but prevents the tool from being embedded in larger processes. |
| **No try‑with‑resources** | Older style; increases risk of leaking streams if an exception occurs. |

### Assumptions & Constraints

* The first command‑line argument is always a valid directory path.  
* The environment has write permission to the target directory.  
* The network connection is available and the remote server responds with the JAR file.  
* No HTTP redirects or authentication are required.  
* The JDK (≥ 7) is available – `java.nio.channels` is used.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| **`main(String[] args)`** | Entry point; orchestrates the download. | `args` – command‑line arguments. | `void` | Prints status, creates directories, exits JVM. | No argument validation; `System.exit` called. |
| **`downloadFileFromURL(String urlString, File destination)`** | Downloads a file from a URL into `destination`. | `urlString` – remote URL; `destination` – file to write. | `void` | Creates/overwrites the file; may throw any `Exception`. | Uses `URL.openStream()`, `ReadableByteChannel`, `FileOutputStream`. No HTTP status checks. |

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `java.net.*` | JDK | Used for `URL` handling. |
| `java.io.*` | JDK | File and stream I/O. |
| `java.nio.channels.*` | JDK | `ReadableByteChannel` & channel transfer. |
| `java.util.Properties` | JDK | Loading properties file. |

All dependencies are part of the standard JDK; no third‑party libraries are required.

---

## 5. Additional Notes & Recommendations

### 5.1 Edge Cases & Missing Robustness

1. **Missing Argument** – `args[0]` will throw an `ArrayIndexOutOfBoundsException`.  
2. **Non‑existent or Non‑Directory Base** – No check; may attempt to write into a file or nonexistent location.  
3. **HTTP Errors** – The code blindly reads the input stream; a 404 or 500 will still create an empty file.  
4. **Redirects** – `URL.openStream()` follows redirects, but the final URL is not logged.  
5. **Resource Leaks** – `ReadableByteChannel` and `FileOutputStream` are closed in a `finally`, but a more modern `try‑with‑resources` would guarantee closure even if `transferFrom` throws.  
6. **Large Files** – `transferFrom` with `Long.MAX_VALUE` is okay for most JARs, but not ideal for very large downloads.  
7. **Thread‑safety** – Not a concern for this single‑threaded tool.  

### 5.2 Suggested Enhancements

| Feature | Implementation Ideas |
|---------|----------------------|
| **Argument Validation** | Check `args.length` and ensure the first argument points to an existing directory. |
| **Property Loading** | Use `try‑with‑resources` for the `FileInputStream`. |
| **HTTP Status Handling** | Use `HttpURLConnection` to fetch the file, inspect `getResponseCode()`, and handle redirects or errors explicitly. |
| **Dynamic Version** | Accept an optional `--version` argument or read `distributionUrl` from the properties to allow updating to newer Maven Wrapper releases. |
| **Logging** | Replace `System.out` with a lightweight logger (e.g., `java.util.logging.Logger`) or allow a `--quiet` flag. |
| **Graceful Exit** | Remove `System.exit` and let the caller decide the exit code; useful if embedding this class. |
| **Progress Reporting** | Hook into `transferFrom` or copy in chunks to report download progress. |
| **Exception Specificity** | Catch `MalformedURLException`, `IOException`, etc., and provide user‑friendly messages. |
| **Unit Tests** | Mock network responses and properties loading to verify behavior under different scenarios. |

### 5.3 Code Refactor Snippet

```java
public static void main(String[] args) {
    if (args.length == 0) {
        System.err.println("Usage: MavenWrapperDownloader <project‑root‑directory>");
        System.exit(1);
    }

    File baseDir = new File(args[0]).getAbsoluteFile();
    if (!baseDir.isDirectory()) {
        System.err.println("Provided path is not a directory: " + baseDir);
        System.exit(1);
    }

    // Load custom URL if present
    String url = DEFAULT_DOWNLOAD_URL;
    File propsFile = new File(baseDir, MAVEN_WRAPPER_PROPERTIES_PATH);
    if (propsFile.exists()) {
        try (FileInputStream fis = new FileInputStream(propsFile)) {
            Properties props = new Properties();
            props.load(fis);
            url = props.getProperty(PROPERTY_NAME_WRAPPER_URL, url);
        } catch (IOException e) {
            System.err.println("Failed to read properties: " + e.getMessage());
        }
    }

    File outFile = new File(baseDir, MAVEN_WRAPPER_JAR_PATH);
    outFile.getParentFile().mkdirs();

    try {
        downloadFileFromURL(url, outFile);
        System.out.println("Downloaded to " + outFile);
    } catch (IOException e) {
        System.err.println("Download failed: " + e.getMessage());
        System.exit(1);
    }
}

private static void downloadFileFromURL(String urlString, File destination) throws IOException {
    HttpURLConnection conn = (HttpURLConnection) new URL(urlString).openConnection();
    conn.setConnectTimeout(15000);
    conn.setReadTimeout(15000);

    if (conn.getResponseCode() != HttpURLConnection.HTTP_OK) {
        throw new IOException("Server returned HTTP " + conn.getResponseCode());
    }

    try (InputStream in = conn.getInputStream();
         OutputStream out = Files.newOutputStream(destination.toPath())) {
        byte[] buffer = new byte[8192];
        int n;
        while ((n = in.read(buffer)) != -1) {
            out.write(buffer, 0, n);
        }
    }
}
```

*The snippet demonstrates proper resource handling, error checks, and a cleaner main method.*

### 5.4 Conclusion

The original code fulfills its narrow purpose and is easy to understand. However, it lacks defensive programming, robust error handling, and modern Java idioms. By adopting the recommendations above, the utility can become more reliable, maintainable, and adaptable to changes in Maven Wrapper releases or deployment environments.

## Code Critique



## Code Preview

```java
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

import java.net.*;
import java.io.*;
import java.nio.channels.*;
import java.util.Properties;

public class MavenWrapperDownloader {

    /**
     * Default URL to download the maven-wrapper.jar from, if no 'downloadUrl' is provided.
     */
    private static final String DEFAULT_DOWNLOAD_URL =
            "https://repo.maven.apache.org/maven2/io/takari/maven-wrapper/0.4.2/maven-wrapper-0.4.2.jar";

    /**
     * Path to the maven-wrapper.properties file, which might contain a downloadUrl property to
     * use instead of the default one.
     */
    private static final String MAVEN_WRAPPER_PROPERTIES_PATH =
            ".mvn/wrapper/maven-wrapper.properties";

    /**
     * Path where the maven-wrapper.jar will be saved to.
     */
    private static final String MAVEN_WRAPPER_JAR_PATH =
            ".mvn/wrapper/maven-wrapper.jar";

    /**
     * Name of the property which should be used to override the default download url for the wrapper.
     */
    private static final String PROPERTY_NAME_WRAPPER_URL = "wrapperUrl";

    public static void main(String args[]) {
        System.out.println("- Downloader started");
        File baseDirectory = new File(args[0]);
        System.out.println("- Using base directory: " + baseDirectory.getAbsolutePath());

        // If the maven-wrapper.properties exists, read it and check if it contains a custom
        // wrapperUrl parameter.
        File mavenWrapperPropertyFile = new File(baseDirectory, MAVEN_WRAPPER_PROPERTIES_PATH);
        String url = DEFAULT_DOWNLOAD_URL;
        if(mavenWrapperPropertyFile.exists()) {
            FileInputStream mavenWrapperPropertyFileInputStream = null;
            try {
                mavenWrapperPropertyFileInputStream = new FileInputStream(mavenWrapperPropertyFile);
                Properties mavenWrapperProperties = new Properties();
                mavenWrapperProperties.load(mavenWrapperPropertyFileInputStream);
                url = mavenWrapperProperties.getProperty(PROPERTY_NAME_WRAPPER_URL, url);
            } catch (IOException e) {
                System.out.println("- ERROR loading '" + MAVEN_WRAPPER_PROPERTIES_PATH + "'");
            } finally {
                try {
                    if(mavenWrapperPropertyFileInputStream != null) {
                        mavenWrapperPropertyFileInputStream.close();
                    }
                } catch (IOException e) {
                    // Ignore ...
                }
            }
        }
        System.out.println("- Downloading from: : " + url);

        File outputFile = new File(baseDirectory.getAbsolutePath(), MAVEN_WRAPPER_JAR_PATH);
        if(!outputFile.getParentFile().exists()) {
            if(!outputFile.getParentFile().mkdirs()) {
                System.out.println(
                        "- ERROR creating output direcrory '" + outputFile.getParentFile().getAbsolutePath() + "'");
            }
        }
        System.out.println("- Downloading to: " + outputFile.getAbsolutePath());
        try {
            downloadFileFromURL(url, outputFile);
            System.out.println("Done");
            System.exit(0);
        } catch (Throwable e) {
            System.out.println("- Error downloading");
            e.printStackTrace();
            System.exit(1);
        }
    }

    private static void downloadFileFromURL(String urlString, File destination) throws Exception {
        URL website = new URL(urlString);
        ReadableByteChannel rbc;
        rbc = Channels.newChannel(website.openStream());
        FileOutputStream fos = new FileOutputStream(destination);
        fos.getChannel().transferFrom(rbc, 0, Long.MAX_VALUE);
        fos.close();
        rbc.close();
    }

}



```
