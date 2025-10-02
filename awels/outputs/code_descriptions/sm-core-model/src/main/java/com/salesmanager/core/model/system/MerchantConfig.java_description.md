# MerchantConfig.java

## Review

## 1. Summary  
**Purpose**  
`MerchantConfig` is a simple value‑object that holds a collection of feature toggles and configuration paths for a merchant‑centric storefront. It is serialisable, can be exposed over REST/JSON and is intended to be populated from a configuration source (database, properties file, etc.).

**Key components**

| Component | Role |
|-----------|------|
| Boolean flags (`displayCustomerSection`, `displayContactUs`, …) | Feature switches controlling UI elements and business behaviour |
| `useDefaultSearchConfig` | Language‑specific boolean switches for whether the default search configuration should be used |
| `defaultSearchConfigPath` | Language‑specific file paths for the default search configuration |
| `toJSONString()` | Serialises the object to a JSON string using `org.json.simple.JSONObject` |

**Design patterns / frameworks**  
- Implements `Serializable` for Java persistence/remote calls.  
- Implements `org.json.simple.JSONAware` to provide a custom JSON serialisation.  
- Uses Apache Commons `StringUtils` for blank‑string checks.  

No sophisticated patterns (e.g., Builder, Strategy) are used; the class is intentionally lightweight.

---

## 2. Detailed Description  
The class is essentially a POJO with two maps and several booleans. The flow of execution is straightforward:

1. **Instantiation** – A new `MerchantConfig` is created, all fields are initialised to default values (`false` or an empty map).
2. **Configuration** – External code calls the provided setters (or a configuration loader) to populate the fields.
3. **Runtime** – The object is used by UI/logic layers to decide whether to display sections, allow purchases, etc.
4. **Serialisation** – When the object needs to be sent over a network or persisted, `toJSONString()` is called:
   * It constructs a new `JSONObject`.
   * Puts each primitive flag using the getters or the field directly.
   * Serialises the two maps, guarding against `null` and blank values.
5. **Cleanup** – No explicit cleanup is required; the class is immutable from the caller’s perspective after construction.

**Assumptions & constraints**

| Assumption | Impact |
|------------|--------|
| The maps will never be `null` (they are initialised in the field declaration). | The `null` checks in `toJSONString()` are redundant but harmless. |
| Keys in the maps are language codes (`en`, `fr`, …). | The class itself does not enforce this; callers must honour the contract. |
| The caller respects the serialisation contract of `JSONAware`. | No versioning or schema validation is performed. |
| The object is thread‑safe only because it is effectively immutable after construction. | If fields are mutated concurrently, race conditions may occur. |

**Architecture**  
The class sits at the bottom of the configuration hierarchy – a thin DTO that is populated by higher‑level services or configuration modules. Because it only holds data and has minimal logic, it can be freely transferred across layers without carrying business rules.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `toJSONString()` | Implements `JSONAware`. Builds a JSON representation of the current state. | None | `String` | None (pure) |
| `setDisplayCustomerSection(boolean)` | Mutator for `displayCustomerSection`. | `boolean` | `void` | Updates internal flag |
| `isDisplayCustomerSection()` | Accessor for `displayCustomerSection`. | None | `boolean` | None |
| `setDisplayContactUs(boolean)` | Mutator for `displayContactUs`. | `boolean` | `void` | Updates internal flag |
| `isDisplayContactUs()` | Accessor for `displayContactUs`. | None | `boolean` | None |
| `setDisplayStoreAddress(boolean)` | Mutator for `displayStoreAddress`. | `boolean` | `void` | Updates internal flag |
| `isDisplayStoreAddress()` | Accessor for `displayStoreAddress`. | None | `boolean` | None |
| `setUseDefaultSearchConfig(Map<String,Boolean>)` | Mutator for the map of language‑specific flags. | `Map<String,Boolean>` | `void` | Replaces the internal map |
| `getUseDefaultSearchConfig()` | Accessor for the map. | None | `Map<String,Boolean>` | Returns reference (potentially mutable) |
| `setDefaultSearchConfigPath(Map<String,String>)` | Mutator for language‑specific path map. | `Map<String,String>` | `void` | Replaces the internal map |
| `getDefaultSearchConfigPath()` | Accessor for the path map. | None | `Map<String,String>` | Returns reference |
| `setDisplayAddToCartOnFeaturedItems(boolean)` | Mutator for `displayAddToCartOnFeaturedItems`. | `boolean` | `void` | Updates flag |
| `isDisplayAddToCartOnFeaturedItems()` | Accessor for that flag. | None | `boolean` | None |
| `setDisplayCustomerAgreement(boolean)` | Mutator for `displayCustomerAgreement`. | `boolean` | `void` | Updates flag |
| `isDisplayCustomerAgreement()` | Accessor for that flag. | None | `boolean` | None |
| `setAllowPurchaseItems(boolean)` | Mutator for `allowPurchaseItems`. | `boolean` | `void` | Updates flag |
| `isAllowPurchaseItems()` | Accessor for that flag. | None | `boolean` | None |
| `setDisplaySearchBox(boolean)` | Mutator for `displaySearchBox`. | `boolean` | `void` | Updates flag |
| `isDisplaySearchBox()` | Accessor for that flag. | None | `boolean` | None |
| `setTestMode(boolean)` | Mutator for `testMode`. | `boolean` | `void` | Updates flag |
| `isTestMode()` | Accessor for `testMode`. | None | `boolean` | None |
| `setDebugMode(boolean)` | Mutator for `debugMode`. | `boolean` | `void` | Updates flag |
| `isDebugMode()` | Accessor for `debugMode`. | None | `boolean` | None |
| `setDisplayPagesMenu(boolean)` | Mutator for `displayPagesMenu`. | `boolean` | `void` | Updates flag |
| `isDisplayPagesMenu()` | Accessor for `displayPagesMenu`. | None | `boolean` | None |

**Reusable / utility methods** – None beyond the standard getters/setters.

---

## 4. Dependencies  

| Library | Purpose | Standard / Third‑Party | Notes |
|---------|---------|------------------------|-------|
| `java.io.Serializable` | Java’s built‑in serialization support | Standard | Provides `serialVersionUID`. |
| `org.json.simple.JSONObject` & `JSONAware` | Manual JSON serialisation | Third‑party (JSON.simple) | Lightweight; no reflection. |
| `org.apache.commons.lang3.StringUtils` | Blank string check (`isBlank`) | Third‑party (Apache Commons Lang) | Avoids `StringUtils.isEmpty` pitfalls. |

No platform‑specific dependencies are present. The class is pure Java and can be used on any JVM.

---

## 5. Additional Notes  

### Edge cases & potential bugs  
| Scenario | Current behaviour | Suggested fix |
|----------|-------------------|---------------|
| `useDefaultSearchConfig` contains `null` values | `toJSONString()` skips those entries. | OK – intended. |
| Maps are mutated after exposure via getters | External code can change internal state unexpectedly. | Return unmodifiable views (`Collections.unmodifiableMap`) or deep copy. |
| `toJSONString()` is called on a partially initialised object (some flags remain default) | JSON will still contain all keys, but values may be `false`. | Acceptable; no explicit validation required. |
| Large maps cause performance overhead during serialisation | Each entry is processed individually. | Cache the JSON representation if the object is immutable for many requests. |
| Null check on maps in `toJSONString()` is redundant | Minor inefficiency. | Remove null checks. |

### Enhancements  

1. **Immutability** – Replace setters with a Builder pattern to create immutable instances.  
2. **Validation** – Add a `validate()` method to ensure language codes follow ISO‑639 format and paths are absolute.  
3. **JSON libraries** – Consider using Jackson or Gson for automatic (de)serialisation; reduces boilerplate.  
4. **Documentation** – Add JavaDoc for each method and describe the expected semantics of the language‑code maps.  
5. **Equals/HashCode** – Override for use in collections or as a key.  
6. **Serialization of maps** – Use `LinkedHashMap` if order matters for readability.  
7. **Logging** – Add optional logging in `toJSONString()` for debugging large configurations.  
8. **Test Mode / Debug Mode** – Provide helper methods that expose behaviour changes when these flags are true.  

Overall, the class is clean, concise, and serves its purpose well. Minor refactors (immutability, unmodifiable map exposure, and modern JSON libraries) would increase robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.system;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.lang3.StringUtils;
import org.json.simple.JSONAware;
import org.json.simple.JSONObject;

public class MerchantConfig implements Serializable, JSONAware {
	

	/**
	 * TODO
	 * Add a generic key value in order to allow the creation of configuration
	 * on the fly from the client application and read from a key value map
	 */
	
	private static final long serialVersionUID = 1L;
	private boolean displayCustomerSection =false;
	private boolean displayContactUs =false;
	private boolean displayStoreAddress = false;
	private boolean displayAddToCartOnFeaturedItems = false;
	private boolean displayCustomerAgreement = false;
	private boolean displayPagesMenu = true;
	private boolean allowPurchaseItems = true;
	private boolean displaySearchBox = true;
	private boolean testMode = false;
	private boolean debugMode = false;
	
	/** Store default search json config **/
	private Map<String,Boolean> useDefaultSearchConfig= new HashMap<String,Boolean>();//language code | true or false
	private Map<String,String> defaultSearchConfigPath= new HashMap<String,String>();//language code | file path

	@SuppressWarnings("unchecked")
	@Override
	public String toJSONString() {
		JSONObject data = new JSONObject();
		data.put("displayCustomerSection", this.isDisplayCustomerSection());
		data.put("displayContactUs", this.isDisplayContactUs());
		data.put("displayStoreAddress", this.isDisplayStoreAddress());
		data.put("displayAddToCartOnFeaturedItems", this.isDisplayAddToCartOnFeaturedItems());
		data.put("displayPagesMenu", this.isDisplayPagesMenu());
		data.put("displayCustomerAgreement", this.isDisplayCustomerAgreement());
		data.put("allowPurchaseItems", this.isAllowPurchaseItems());
		data.put("displaySearchBox", this.displaySearchBox);
		data.put("testMode", this.isTestMode());
		data.put("debugMode", this.isDebugMode());
		
		if(useDefaultSearchConfig!=null) {
			JSONObject obj = new JSONObject();
			for(String key : useDefaultSearchConfig.keySet()) {
				Boolean val = (Boolean)useDefaultSearchConfig.get(key);
				if(val!=null) {
					obj.put(key,val);
				}
			}
			data.put("useDefaultSearchConfig", obj);
		}
		
		if(defaultSearchConfigPath!=null) {
			JSONObject obj = new JSONObject();
			for(String key : defaultSearchConfigPath.keySet()) {
				String val = (String)defaultSearchConfigPath.get(key);
				if(!StringUtils.isBlank(val)) {
					obj.put(key, val);
				}
			}
			data.put("defaultSearchConfigPath", obj);
		}
		
		
		return data.toJSONString();
	}

	public void setDisplayCustomerSection(boolean displayCustomerSection) {
		this.displayCustomerSection = displayCustomerSection;
	}

	public boolean isDisplayCustomerSection() {
		return displayCustomerSection;
	}

	public void setDisplayContactUs(boolean displayContactUs) {
		this.displayContactUs = displayContactUs;
	}

	public boolean isDisplayContactUs() {
		return displayContactUs;
	}

	public boolean isDisplayStoreAddress() {
		return displayStoreAddress;
	}

	public void setDisplayStoreAddress(boolean displayStoreAddress) {
		this.displayStoreAddress = displayStoreAddress;
	}

	public void setUseDefaultSearchConfig(Map<String,Boolean> useDefaultSearchConfig) {
		this.useDefaultSearchConfig = useDefaultSearchConfig;
	}

	public Map<String,Boolean> getUseDefaultSearchConfig() {
		return useDefaultSearchConfig;
	}

	public void setDefaultSearchConfigPath(Map<String,String> defaultSearchConfigPath) {
		this.defaultSearchConfigPath = defaultSearchConfigPath;
	}

	public Map<String,String> getDefaultSearchConfigPath() {
		return defaultSearchConfigPath;
	}

	public void setDisplayAddToCartOnFeaturedItems(
			boolean displayAddToCartOnFeaturedItems) {
		this.displayAddToCartOnFeaturedItems = displayAddToCartOnFeaturedItems;
	}

	public boolean isDisplayAddToCartOnFeaturedItems() {
		return displayAddToCartOnFeaturedItems;
	}

	public boolean isDisplayCustomerAgreement() {
		return displayCustomerAgreement;
	}

	public void setDisplayCustomerAgreement(boolean displayCustomerAgreement) {
		this.displayCustomerAgreement = displayCustomerAgreement;
	}

	public boolean isAllowPurchaseItems() {
		return allowPurchaseItems;
	}

	public void setAllowPurchaseItems(boolean allowPurchaseItems) {
		this.allowPurchaseItems = allowPurchaseItems;
	}

	public boolean isDisplaySearchBox() {
		return displaySearchBox;
	}

	public void setDisplaySearchBox(boolean displaySearchBox) {
		this.displaySearchBox = displaySearchBox;
	}

	public boolean isTestMode() {
		return testMode;
	}

	public void setTestMode(boolean testMode) {
		this.testMode = testMode;
	}

	public boolean isDebugMode() {
		return debugMode;
	}

	public void setDebugMode(boolean debugMode) {
		this.debugMode = debugMode;
	}

	public boolean isDisplayPagesMenu() {
		return displayPagesMenu;
	}

	public void setDisplayPagesMenu(boolean displayPagesMenu) {
		this.displayPagesMenu = displayPagesMenu;
	}

}



```
