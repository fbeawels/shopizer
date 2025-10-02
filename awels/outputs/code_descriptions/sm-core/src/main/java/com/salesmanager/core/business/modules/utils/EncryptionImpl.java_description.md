# EncryptionImpl.java

## Review

## 1. Summary  

The `EncryptionImpl` class implements a simple AES‑CBC encryption/decryption utility.  
* **Purpose** – Convert plaintext strings to a hex‑encoded cipher text and back again.  
* **Key components**  
  * `encrypt(String)` – Encrypts a string using AES‑CBC with PKCS5 padding.  
  * `decrypt(String)` – Decrypts a hex‑encoded string back to plaintext.  
  * `bytesToHex` / `hexToBytes` – Helper methods for converting between binary data and a hex string.  
  * `secretKey` – The user‑supplied key used for both encryption and decryption.  
* **Design patterns & libraries** – The class is a plain implementation of a small utility interface (`Encryption`). It relies on the JCA (`Cipher`, `SecretKeySpec`, `IvParameterSpec`) and Apache Commons Lang (`StringUtils`). No advanced frameworks or patterns are used.

---

## 2. Detailed Description  

### Execution Flow  

| Step | Method | Action |
|------|--------|--------|
| 1 | `encrypt` | Build `Cipher` with AES/CBC/PKCS5Padding, initialise with the fixed IV (`IV_P`) and the provided key, then `doFinal` on the raw bytes of the input string. |
| 2 | `bytesToHex` | Convert the resulting byte array into a lowercase hex string. |
| 3 | `decrypt` | Build the same `Cipher`, initialise it for decryption, and `doFinal` on the decoded bytes from the hex string. |
| 4 | Result | Return the plain‑text string (or throw if the input is blank). |

### Assumptions & Constraints  

* **Secret key** – Must be exactly 16 bytes (128‑bit AES) because the code never checks length.  
* **IV** – A static, hard‑coded 16‑byte IV (`fedcba9876543210`).  
* **Charset** – Uses the platform default charset when converting between `String` and `byte[]`.  
* **Security model** – No integrity/authentication check (no MAC or authenticated‑cipher mode).  

### Architecture & Design Choices  

* The implementation is intentionally minimal and focused on string‑to‑hex conversions.  
* It follows a *functional* style rather than a *stateful* service (except for the `secretKey` field).  
* No dependency injection or factory pattern is used; the key is set via a simple setter.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `encrypt(String)` | Encrypts a plain text string. | `String value` – plaintext. | `String` – hex‑encoded ciphertext. | Uses the instance’s `secretKey`. |
| `decrypt(String)` | Decrypts a hex‑encoded ciphertext. | `String value` – ciphertext. | `String` – plaintext. | Throws if input blank. |
| `bytesToHex(byte[])` | Helper: binary to hex string. | `byte[] data` | `String` (hex). | None. |
| `hexToBytes(String)` | Helper: hex to binary. | `String str` | `byte[]` | None. |
| `getSecretKey()` | Getter for `secretKey`. | None | `String` | None. |
| `setSecretKey(String)` | Setter for `secretKey`. | `String secretKey` | None | Updates internal key. |

> **Reusable utilities** – `bytesToHex` and `hexToBytes` can be extracted into a small, static helper class if the project grows.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.crypto.*` | JDK | Standard Java Cryptography Architecture. |
| `org.apache.commons.lang3.StringUtils` | Third‑party | Apache Commons Lang (utility). |
| `com.salesmanager.core.modules.utils.Encryption` | In‑project | Interface that this class implements. |

*No external crypto libraries* beyond JCA.  
*Platform‑specific* – None; uses only standard JDK APIs.

---

## 5. Additional Notes & Recommendations  

### 5.1 Security Concerns  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Static IV** | Predictable IV leads to ciphertext malleability and leaks patterns in the plaintext. | Generate a random IV per encryption, prepend it to the ciphertext (or use an authenticated‑cipher mode). |
| **No authentication** | An attacker can modify the ciphertext and the decryption will succeed with garbage output. | Use an authenticated mode such as AES/GCM/NoPadding or append an HMAC over the ciphertext. |
| **Plain key handling** | Storing or passing the key as a mutable `String` may keep it in memory longer than necessary. | Store the key in a `SecretKeySpec` or `javax.crypto.SecretKey` immediately after validation, and wipe the original string if possible. |
| **Key size not validated** | If the key is shorter or longer than 16 bytes, the `SecretKeySpec` will silently truncate or pad, producing unpredictable results. | Validate `secretKey.length() == 16` (or 24/32 for AES‑192/256) and throw a clear exception otherwise. |
| **Default charset** | `value.getBytes()` and `new String(outText)` use the platform default charset, which may differ between JVMs, breaking round‑trips. | Use a fixed charset such as `StandardCharsets.UTF_8`. |
| **Plaintext length** | PKCS5Padding is fine for most lengths, but the code previously tried NoPadding and padded manually; the current code never checks for block‑size boundaries. | Keep PKCS5/PKCS7 padding but clearly document that the algorithm supports any length. |
| **Exception handling** | `decrypt` throws a generic `Exception` with a misleading message ("Nothing to encrypt") when the input is blank. | Throw a custom `IllegalArgumentException("Ciphertext cannot be null or empty")`. |

### 5.2 Performance & Code Quality  

* **String concatenation in `bytesToHex`** – Uses `String` concatenation in a loop (O(n²) time). Replace with `StringBuilder`.  
* **Redundant null/length checks** – `hexToBytes` returns `null` for empty or single‑char strings; callers should handle this.  
* **Missing Javadoc** – Adding method‑level documentation would improve maintainability.  
* **Magic strings** – `IV_P`, `KEY_SPEC`, `CYPHER_SPEC` should be constants with clear names; the IV string is not hex‑encoded, but plain text.  
* **Thread‑safety** – The class is not thread‑safe because `secretKey` can be mutated while encrypt/decrypt runs. Mark the class immutable or synchronize access.

### 5.3 Potential Enhancements  

1. **Switch to AES/GCM** – Provides confidentiality + integrity in a single operation.  
2. **Key derivation** – Use PBKDF2/SCrypt/Argon2 to derive a key from a password.  
3. **Key/IV storage** – Move key handling to a secure keystore or environment variable; avoid setting via plain setter.  
4. **API simplification** – Provide overloaded methods that accept/return byte arrays directly for binary data.  
5. **Unit tests** – Add deterministic tests for encryption/decryption, key validation, and error handling.

---

### Bottom Line  

`EncryptionImpl` delivers a straightforward AES‑CBC encrypt/decrypt pair suitable for very lightweight use cases. However, its current design is **insecure** for production due to the fixed IV, lack of authentication, and poor key handling. It also exhibits some code‑quality issues (charset, string concatenation, exception messaging). For any application that requires strong cryptographic guarantees, refactor to use an authenticated cipher mode (e.g., AES/GCM) and enforce strict key/IV management. If the goal is just to have a quick “string obfuscator”, then document the limitations clearly and consider wrapping the implementation in a higher‑level service that validates configuration before use.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.utils;

import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;

import org.apache.commons.lang3.StringUtils;

import com.salesmanager.core.modules.utils.Encryption;

public final class EncryptionImpl implements Encryption {
	
	private final static String IV_P = "fedcba9876543210";
	private final static String KEY_SPEC = "AES";
	private final static String CYPHER_SPEC = "AES/CBC/PKCS5Padding";
	


    private String  secretKey;



	@Override
	public String encrypt(String value) throws Exception {

		
		// value = StringUtils.rightPad(value, 16,"*");
		// Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
		// NEED TO UNDERSTAND WHY PKCS5Padding DOES NOT WORK
		Cipher cipher = Cipher.getInstance(CYPHER_SPEC);
		SecretKeySpec keySpec = new SecretKeySpec(secretKey.getBytes(), KEY_SPEC);
		IvParameterSpec ivSpec = new IvParameterSpec(IV_P
				.getBytes());
		cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);
		byte[] inpbytes = value.getBytes();
		byte[] encrypted = cipher.doFinal(inpbytes);
		return bytesToHex(encrypted);
		
		
	}

	@Override
	public String decrypt(String value) throws Exception {

		
		if (StringUtils.isBlank(value))
			throw new Exception("Nothing to encrypt");

		// NEED TO UNDERSTAND WHY PKCS5Padding DOES NOT WORK
		// Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
		Cipher cipher = Cipher.getInstance(CYPHER_SPEC);
		SecretKeySpec keySpec = new SecretKeySpec(secretKey.getBytes(), KEY_SPEC);
		IvParameterSpec ivSpec = new IvParameterSpec(IV_P
				.getBytes());
		cipher.init(Cipher.DECRYPT_MODE, keySpec, ivSpec);
		byte[] outText;
		outText = cipher.doFinal(hexToBytes(value));
		return new String(outText);
		
		
	}
	
	
	private String bytesToHex(byte[] data) {
		if (data == null) {
			return null;
		} else {
			int len = data.length;
			String str = "";
			for (byte datum : data) {
				if ((datum & 0xFF) < 16) {
					str = str + "0"
							+ Integer.toHexString(datum & 0xFF);
				} else {
					str = str + Integer.toHexString(datum & 0xFF);
				}

			}
			return str;
		}
	}

	private static byte[] hexToBytes(String str) {
		if (str == null) {
			return null;
		} else if (str.length() < 2) {
			return null;
		} else {
			int len = str.length() / 2;
			byte[] buffer = new byte[len];
			for (int i = 0; i < len; i++) {
				buffer[i] = (byte) Integer.parseInt(str.substring(i * 2,
						i * 2 + 2), 16);
			}
			return buffer;
		}
	}
	
	public String getSecretKey() {
		return secretKey;
	}

	public void setSecretKey(String secretKey) {
		this.secretKey = secretKey;
	}

}



```
