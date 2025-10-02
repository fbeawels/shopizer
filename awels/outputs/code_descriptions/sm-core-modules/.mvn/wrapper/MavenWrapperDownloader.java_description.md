# MavenWrapperDownloader.java

## Review

## 1. Summary

The **`MavenWrapperDownloader`** is a tiny, self‑contained Java utility that downloads the Maven Wrapper JAR (`maven-wrapper.jar`) into a given project directory.  
- **Purpose** – Ensure a project contains the correct Maven Wrapper JAR even if it is missing or corrupted.  
- **Key components**  
  - `DEFAULT_DOWNLOAD_URL` – Fallback location for the JAR.  
  - `MAVEN_WRAPPER_PROPERTIES_PATH` – Optional properties file that can override the download URL.  
  - `downloadFileFromURL` – Core routine that streams the remote file to disk.  
- **Design** – Procedural, single‑class implementation with no external dependencies beyond the JDK.  
- **Patterns** – The code follows a simple “configuration overrides default” pattern and uses a classic stream‑copy pattern for downloading.

---

## 2. Detailed Description

### Flow of Execution

1. **Argument handling**  
   - Expects a single command‑line argument: the base directory of a Maven project.  
   - Prints the absolute path of this directory.

2. **Configuration lookup**  
   - Looks for `./.mvn/wrapper/maven-wrapper.properties` inside the base directory.  
   - If present, loads it and reads the `wrapperUrl` property; otherwise the hard‑coded default URL is used.

3. **Directory preparation**  
   - Computes the target path `./.mvn/wrapper/maven-wrapper.jar`.  
   - Creates any missing parent directories (throws a console error if the directory cannot be created).

4. **Download**  
   - Calls `downloadFileFromURL` with the resolved URL and destination file.  
   - On success, prints “Done” and exits with code 0; on any exception, prints an error, dumps the stack trace, and exits with code 1.

### Dependencies & Constraints

- **JDK classes only** – `java.net`, `java.io`, `java.nio.channels`, `java.util.Properties`.  
- **Assumes**:  
  - The first command‑line argument exists and is a valid directory.  
  - The `wrapperUrl` property, if present, is a well‑formed HTTP/HTTPS URL.  
  - The program has write permissions to the target directory.  
- **No multithreading or network retries** – a single, blocking download.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `public static void main(String[] args)` | Entry point; orchestrates argument parsing, configuration loading, and file download. | `String[] args` – command‑line arguments. | `void` | Console output; creates directories; writes file; exits process. |
| `private static void downloadFileFromURL(String urlString, File destination)` | Streams a remote resource into a local file. | `String urlString` – download URL.<br> `File destination` – where to store the JAR. | `void` | Opens network channel, writes to disk, closes streams. |

- **Reusable utility** – `downloadFileFromURL` could be extracted to a shared helper if more download logic is needed elsewhere.

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `java.net.URL` | Standard JDK | For parsing the download URL. |
| `java.io.*` | Standard JDK | File I/O, stream handling. |
| `java.nio.channels.*` | Standard JDK | Efficient channel‑to‑channel transfer. |
| `java.util.Properties` | Standard JDK | Reads the optional wrapper properties file. |

No external frameworks or third‑party libraries are used.

---

## 5. Additional Notes

### Strengths
- **Simplicity** – minimal code, easy to understand and maintain.  
- **Self‑contained** – can run from any JDK without extra setup.  
- **Configurable** – respects a custom `wrapperUrl` property.

### Weaknesses & Edge Cases
1. **Missing argument** – `args[0]` access without bounds checking causes `ArrayIndexOutOfBoundsException` if no arguments are supplied.  
2. **Hard‑coded URL** – The default URL (`0.4.2`) may become stale; a build script would need to update the constant manually.  
3. **No retry logic** – A transient network failure aborts the program immediately.  
4. **No progress indicator** – Large downloads produce no user feedback.  
5. **Message typo** – `System.out.println("- Downloading from: : " + url);` prints a double colon.  
6. **Resource leaks** – While the code does close streams, it uses manual `close()` calls; a try‑with‑resources block would be safer and more concise.  
7. **Overwrite policy** – Existing `maven-wrapper.jar` is unconditionally overwritten; no check for up‑to‑date version.  
8. **Encoding/Whitespace** – The property value is not trimmed; leading/trailing spaces could break the URL.  
9. **Security** – No validation of the URL scheme (e.g., only HTTP/HTTPS allowed).  
10. **Cross‑platform path handling** – Relies on `File.separator` implicitly via `new File(base, path)`; works on Windows and Unix but still uses literal `.` and `/` in constants, which are fine for Java but worth noting.

### Suggested Enhancements
- **Argument validation** – Check `args.length >= 1` and print a usage message if missing.  
- **Graceful exit codes** – Use a dedicated `exit` method to centralise exit logic.  
- **Try‑with‑resources** – Simplify stream handling and guarantee closure.  
- **Download progress** – Hook into `transferFrom` progress or use a buffered read loop.  
- **Retry/backoff** – Attempt a few downloads on failure before giving up.  
- **Version detection** – Compare the downloaded JAR’s version (e.g., via `MANIFEST.MF`) with the local one to avoid unnecessary overwrites.  
- **Configuration injection** – Allow the default URL and property path to be overridden via system properties or environment variables.  
- **Logging framework** – Replace `System.out.println` with SLF4J or `java.util.logging` for better log management.  

Overall, the utility accomplishes its narrow goal correctly but would benefit from defensive coding and modern Java idioms for robustness and maintainability.

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
