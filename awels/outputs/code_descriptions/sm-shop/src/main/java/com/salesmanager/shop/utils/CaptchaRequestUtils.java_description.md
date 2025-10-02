# CaptchaRequestUtils.java

## Review

## 1. Summary  

**Purpose**  
`CaptchaRequestUtils` is a Spring‑managed component that validates a Google reCAPTCHA v2 response token by calling the public verification API (`https://www.google.com/recaptcha/api/siteverify`). It encapsulates the HTTP call, JSON parsing and basic error handling, exposing a single `checkCaptcha(String)` method that returns `true` if the token is verified, otherwise `false` or throws an exception.

**Key components**

| Component | Role |
|-----------|------|
| `CoreConfiguration` | Reads configuration values (in this case the reCAPTCHA URL). |
| `@Value("${config.recaptcha.secretKey}")` | Injects the reCAPTCHA secret key. |
| `HttpClient` / `HttpPost` | Performs the HTTP POST to the verification endpoint. |
| Jackson `ObjectMapper` | Deserialises the JSON response. |
| `checkCaptcha` | Public API that orchestrates the call and returns the result. |

**Design patterns & libraries**

* **Builder Pattern** – `HttpClientBuilder.create().build()` constructs the HTTP client.
* **Dependency Injection** – Spring’s `@Component` and `@Value` annotations.
* **Third‑party libs** – Apache HttpComponents (client 4.x), Jackson, Apache Commons Lang, Spring Framework.

---

## 2. Detailed Description  

### Flow of execution

1. **Injection** – On application startup, Spring injects `CoreConfiguration` and the `secretKey` value.
2. **Invocation** – A consumer calls `checkCaptcha(gRecaptchaResponse)`.
3. **HTTP client creation** – A new `HttpClient` instance is built each time (`HttpClientBuilder.create().build()`).
4. **Request construction** –  
   * URL is retrieved via `configuration.getProperty(ApplicationConstants.RECAPTCHA_URL)`.  
   * Form parameters `secret` and `response` are added to an `ArrayList<NameValuePair>`.  
   * A `HttpPost` object is created and populated with an `UrlEncodedFormEntity`.
5. **Execution & response handling** –  
   * `client.execute(post)` returns an `HttpResponse`.  
   * Status code is checked; anything other than 200 throws a generic `Exception`.  
   * The response entity is read into a byte array and converted to a UTF‑8 `String`.  
   * Jackson parses the JSON into a `Map<String,String>`.  
   * The `success` key is extracted; a blank value also triggers an exception.  
   * The string value is converted to a `Boolean`; if `true` the method returns `true`, otherwise `false`.
6. **Cleanup** – `post.releaseConnection()` is called in a `finally` block.

### Assumptions & constraints

| Assumption | Rationale |
|------------|-----------|
| The response JSON contains a top‑level `"success"` field with a boolean string (`"true"`/`"false"`). | This matches reCAPTCHA v2. |
| The request always succeeds within the default connection timeouts. | No explicit timeout configuration is provided. |
| The `secretKey` and URL are always non‑null and correctly injected. | No null checks for these values. |

### Architecture & design choices

* The class is a **utility component** focused on a single responsibility – verifying a captcha token.  
* It uses **imperative HTTP client** code instead of higher‑level abstractions (e.g., Spring `RestTemplate` or `WebClient`).  
* Error handling is **exception‑heavy**; any HTTP or parsing error throws a raw `Exception`.  
* JSON parsing is performed via a generic `Map`, rather than a strongly‑typed DTO.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `public boolean checkCaptcha(String gRecaptchaResponse) throws Exception` | Validates the reCAPTCHA token by contacting Google’s verification endpoint. | `String gRecaptchaResponse` – the client’s token | `boolean` – `true` if verified, `false` otherwise | • Throws `Exception` on network, HTTP, or parsing errors. <br>• Creates a new `HttpClient` per call (resource not explicitly closed). <br>• Releases the HTTP connection via `post.releaseConnection()`. |

**Reusable/utility methods** – None; the logic is encapsulated entirely in `checkCaptcha`.

---

## 4. Dependencies  

| Library | Usage | Standard / 3rd‑party |
|---------|-------|-----------------------|
| **Spring Framework** (`@Component`, `@Value`) | Dependency injection & configuration | Third‑party (Spring) |
| **Apache HttpComponents** (`HttpClient`, `HttpPost`, etc.) | Low‑level HTTP communication | Third‑party |
| **Jackson** (`ObjectMapper`, `TypeReference`) | JSON deserialization | Third‑party |
| **Apache Commons Lang** (`StringUtils`) | String utilities | Third‑party |
| **Java SE** (`javax.inject.Inject`, `java.util`, `java.nio.charset.StandardCharsets`) | Core language features | Standard |

No platform‑specific code is present; the component can run on any JVM that satisfies the library versions.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Missing or malformed configuration** – No checks for `null` or empty `secretKey` or URL.  
2. **Network timeouts** – The default `HttpClient` may block indefinitely if Google is unreachable.  
3. **Resource leakage** – A new `HttpClient` is created each call and never closed. While `post.releaseConnection()` frees the underlying connection, the client instance itself holds resources (e.g., thread pools) that accumulate over time.  
4. **Unsupported reCAPTCHA versions** – The implementation only handles the boolean `success` field; v3 responses contain `score` and `action` that are ignored.  
5. **Exception granularity** – Throwing raw `Exception` obscures the cause; callers cannot differentiate between network errors, invalid JSON, or application‑level failures.  

### Potential Enhancements  

| Enhancement | Benefit |
|-------------|---------|
| **Reuse a single `HttpClient`** (e.g., injected singleton) | Reduces overhead and frees thread pools early. |
| **Switch to Spring’s `RestTemplate` or `WebClient`** | Simplifies code, handles timeouts, and integrates with Spring’s exception handling. |
| **Introduce a DTO (`RecaptchaResponse`)** | Stronger typing, clearer mapping, easier future extensions. |
| **Add timeout configuration** | Prevents long‑hanging calls. |
| **Create custom exception hierarchy** (`CaptchaValidationException`, `CaptchaNetworkException`, etc.) | Allows callers to handle specific failure modes. |
| **Support reCAPTCHA v3** | Parse `score` and `action` for advanced validation logic. |
| **Input validation** (`StringUtils.isBlank(gRecaptchaResponse)`) | Avoid unnecessary HTTP calls. |
| **Unit tests with a mock HTTP server** | Ensure reliability of parsing logic and error handling. |

### Security Considerations  

* The `secretKey` is injected from application properties; ensure it is stored securely (e.g., encrypted config, secrets manager).  
* The code exposes the raw `Exception` messages; avoid leaking sensitive details in production logs.  

---

### Bottom‑line  

`CaptchaRequestUtils` delivers the core functionality of reCAPTCHA v2 validation in a concise, self‑contained component. However, its current implementation has several areas that can be improved for robustness, maintainability, and performance. Adopting a higher‑level HTTP client abstraction, adding proper exception handling, and ensuring resource reuse will make the component more production‑ready.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.utils;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.salesmanager.core.business.utils.CoreConfiguration;
import com.salesmanager.shop.constants.ApplicationConstants;
import org.apache.commons.lang3.StringUtils;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.HttpStatus;
import org.apache.http.NameValuePair;
import org.apache.http.client.HttpClient;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.message.BasicNameValuePair;
import org.apache.http.util.EntityUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.inject.Inject;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Creates a request to reCaptcha 2
 * https://www.google.com/recaptcha/api/siteverify
 * Throws an exception if it can't connect to reCaptcha
 * returns true or false if validation has passed
 * @author carlsamson
 *
 */
@Component
public class CaptchaRequestUtils {
	
	@Inject
	private CoreConfiguration configuration; //for reading public and secret key
	
	private static final String SUCCESS_INDICATOR = "success";
	
	  @Value("${config.recaptcha.secretKey}")
	  private String secretKey;
	
	public boolean checkCaptcha(String gRecaptchaResponse) throws Exception {

		HttpClient client = HttpClientBuilder.create().build();
	    
	    String url = configuration.getProperty(ApplicationConstants.RECAPTCHA_URL);;

        List<NameValuePair> data = new ArrayList<NameValuePair>();
        data.add(new BasicNameValuePair("secret",  secretKey));
        data.add(new BasicNameValuePair("response",  gRecaptchaResponse));

	    
	    // Create a method instance.
        HttpPost post = new HttpPost(url);
	    post.setEntity(new UrlEncodedFormEntity(data,StandardCharsets.UTF_8));
	    
	    boolean checkCaptcha = false;
	    

	    try {
	      // Execute the method.
            HttpResponse httpResponse = client.execute(post);
            int statusCode = httpResponse.getStatusLine().getStatusCode();

	      if (statusCode != HttpStatus.SC_OK) {
	    	throw new Exception("Got an invalid response from reCaptcha " + url + " [" + httpResponse.getStatusLine() + "]");
	      }

	      // Read the response body.
            HttpEntity entity = httpResponse.getEntity();
            byte[] responseBody =EntityUtils.toByteArray(entity);


	      // Deal with the response.
	      // Use caution: ensure correct character encoding and is not binary data
	      //System.out.println(new String(responseBody));
	      
	      String json = new String(responseBody);
	      
	      Map<String,String> map = new HashMap<String,String>();
	  	  ObjectMapper mapper = new ObjectMapper();
	  	  
	  	  map = mapper.readValue(json, 
			    new TypeReference<HashMap<String,String>>(){});
	  	  
	  	  String successInd = map.get(SUCCESS_INDICATOR);
	  	  
	  	  if(StringUtils.isBlank(successInd)) {
	  		  throw new Exception("Unreadable response from reCaptcha " + json);
	  	  }
	  	  
	  	  Boolean responseBoolean = Boolean.valueOf(successInd);
	  	  
	  	  if(responseBoolean) {
	  		checkCaptcha = true;
	  	  }
	  	  
	  	  return checkCaptcha;

	    } finally {
	      // Release the connection.
	      post.releaseConnection();
	    }  
	  }


}



```
