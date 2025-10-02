# ShippingConfiguration.java

## Review

## 1. Summary  

`ShippingConfiguration` is a persistence‑ready Java POJO that represents a single shipping rule set.  
It holds everything needed to calculate shipping costs (package dimensions, weight limits, fee types, free‑shipping thresholds, etc.) and can be serialised to/from JSON through the `org.json.simple` library.  

Key components  

| Component | Purpose | Remarks |
|-----------|---------|---------|
| **Enums** (`ShippingType`, `ShippingBasisType`, `ShippingOptionPriceType`, `ShippingPackageType`, `ShippingDescription`) | Define allowed values for the various configuration fields | External to this class – must be available at compile time |
| **JSON binding fields** (`shipType`, `shipBaseType`, …) | Mirrors the enum values as Strings for easy JSON round‑tripping | Duplicates internal enum state – risk of desynchronisation |
| **Package list** (`List<Package>`) | Collection of nested package objects (likely a custom class) | Uses the name `Package` which shadows `java.lang.Package` |
| **BigDecimal fields** (`handlingFees`, `orderTotalFreeShipping`) | Store monetary values | Nullability is not protected |
| **toJSONString** | Serialises the object to JSON | Manual mapping – no external library support |

The design is fairly conventional: mutable, JavaBean‑style with a lot of getters/setters and a custom JSON converter. No advanced frameworks (e.g., Jackson, Hibernate) are used; the class is deliberately lightweight.

---

## 2. Detailed Description  

### 2.1 Initialization  

The constructor is implicit (the default no‑arg constructor). All fields are initialised to default Java values (null/0/false).  
When a `ShippingConfiguration` instance is populated from JSON, the *JSON binding* setters (`setShipType`, `setShipBaseType`, …) are invoked. Each of these methods:

1. Stores the incoming String value in the corresponding `shipX` field.
2. Parses that string into the appropriate enum and stores it in the “real” field (`shippingType`, `shippingBasisType`, …).

This two‑step process keeps the object in sync with the JSON representation but introduces duplicate state.

### 2.2 Runtime behaviour  

During normal use the following operations are expected:

| Operation | Typical flow |
|-----------|--------------|
| **Reading configuration** | Call getters (`getShippingType()`, `getBoxWidth()`, etc.) – simply return the stored values. |
| **Calculating shipping cost** | The caller will examine the configuration fields and apply business logic (not present in this class). |
| **Persisting to JSON** | `toJSONString()` is called. It creates a `JSONObject`, populates it with the current state, serialises nested `Package` objects via `transformPackage()`, and returns the JSON string. |

### 2.3 Cleanup  

No explicit cleanup is required – all resources are primitive or managed by the JVM.

### 2.4 Assumptions & Constraints  

* All enum values are serialised using their `name()` – any change to the enum name will break backward compatibility.  
* The JSON binding methods assume non‑null input strings; a `null` value will throw `NullPointerException`.  
* No validation is performed on numeric values (e.g., negative dimensions or weights are allowed).  
* The class is *not* thread‑safe; concurrent updates may corrupt state.

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `getShipType()` | Returns the JSON‑friendly string of the shipping type. | – | `String` | – |
| `getShipBaseType()` | Same for shipping basis type. | – | `String` | – |
| `getShipOptionPriceType()` | Same for shipping option price type. | – | `String` | – |
| `setShippingOptionPriceType(ShippingOptionPriceType)` | Updates enum field & JSON string. | `shippingOptionPriceType` | – | `this.shippingOptionPriceType`, `this.shipOptionPriceType` |
| `getShippingOptionPriceType()` | Returns enum. | – | `ShippingOptionPriceType` | – |
| `setShippingBasisType(ShippingBasisType)` | Updates enum & JSON string. | `shippingBasisType` | – | `this.shippingBasisType`, `this.shipBaseType` |
| `getShippingBasisType()` | Returns enum. | – | `ShippingBasisType` | – |
| `setShippingType(ShippingType)` | Updates enum & JSON string. | `shippingType` | – | `this.shippingType`, `this.shipType` |
| `getShippingType()` | Returns enum. | – | `ShippingType` | – |
| `getShippingPackageType()` / `setShippingPackageType(ShippingPackageType)` | Get/set enum for package type. | – / `shippingPackageType` | `ShippingPackageType` | updates `shipPackageType` |
| `getShipPackageType()` | Returns JSON string of package type. | – | `String` | – |
| `setShipType(String)` | JSON‑binding setter – parses string to enum. | `shipType` | – | updates both fields |
| `setShipOptionPriceType(String)` | Same for option price type. | `shipOptionPriceType` | – | – |
| `setShipBaseType(String)` | Same for basis type. | `shipBaseType` | – | – |
| `setShipPackageType(String)` | Same for package type. | `shipPackageType` | – | – |
| `setShipDescription(String)` | Sets description string and enum. | `shipDescription` | – | – |
| `setShipFreeType(String)` | Sets free shipping type string and enum. | `shipFreeType` | – | – |
| `toJSONString()` | Serialises the configuration to JSON. | – | `String` | – |
| `transformPackage(Package)` | Helper for serialising a nested `Package` object. | `p` | `JSONObject` | – |
| `getBoxWidth()/setBoxWidth(int)` | Get/set package width. | – / `boxWidth` | `int` | – |
| `getBoxHeight()/setBoxHeight(int)` | Get/set height. | – / `boxHeight` | `int` | – |
| `getBoxLength()/setBoxLength(int)` | Get/set length. | – / `boxLength` | `int` | – |
| `getBoxWeight()/setBoxWeight(double)` | Get/set box weight. | – / `boxWeight` | `double` | – |
| `getMaxWeight()/setMaxWeight(double)` | Get/set max weight. | – / `maxWeight` | `double` | – |
| `isFreeShippingEnabled()/setFreeShippingEnabled(boolean)` | Flag for free shipping. | – / `freeShippingEnabled` | `boolean` | – |
| `getOrderTotalFreeShipping()/setOrderTotalFreeShipping(BigDecimal)` | Free‑shipping threshold. | – / `orderTotalFreeShipping` | `BigDecimal` | – |
| `setHandlingFees(BigDecimal)` / `getHandlingFees()` | Shipping handling fee. | – / `handlingFees` | `BigDecimal` | – |
| `setTaxOnShipping(boolean)` / `isTaxOnShipping()` | Flag for tax. | – / `taxOnShipping` | `boolean` | – |
| `getShipDescription()/setShippingDescription(ShippingDescription)` | Description enum getter/setter. | – / `shippingDescription` | `String` / `ShippingDescription` | – |
| `getShipFreeType()/setFreeShippingType(ShippingType)` | Free‑shipping enum getter/setter. | – / `freeShippingType` | `String` / `ShippingType` | – |
| `setOrderTotalFreeShippingText(String)` / `getOrderTotalFreeShippingText()` | Transient string representation (UI helper). | – / `orderTotalFreeShippingText` | – | – |
| `setHandlingFeesText(String)` / `getHandlingFeesText()` | Transient string representation (UI helper). | – / `handlingFeesText` | – | – |
| `getPackages()/setPackages(List<Package>)` | Get/set nested packages list. | – / `packages` | `List<Package>` | – |

*Utility methods*: None beyond `transformPackage()`.

---

## 4. Dependencies  

| Library | Version (assumed) | Role |
|---------|------------------|------|
| **json-simple** (`org.json.simple`) | Any 1.x version | Provides `JSONObject`, `JSONArray`, and the `JSONAware` interface used for manual JSON conversion |
| **Java SE** | 1.8+ | Core language features (`BigDecimal`, generics, etc.) |

No other third‑party frameworks are required. All enums and the `Package` class are expected to be part of the same project.

---

## 5. Additional Notes & Recommendations  

### 5.1 Design Issues  
1. **Duplicate state** – The class stores both enum values and their `String` representations. The JSON binding setters maintain both, but any direct manipulation of the enum field can desynchronise the string field (and vice‑versa).  
   *Recommendation:* Keep a single source of truth (prefer the enum) and generate the JSON string on‑the‑fly in `toJSONString()`.

2. **Naming inconsistencies** – Methods such as `getShipPackageType()` vs. `getShippingPackageType()` and `getShipDescription()` vs. `getShippingDescription()` create confusion.  
   *Recommendation:* Adopt a consistent prefix (`shipping`) for all getters/setters.

3. **Shadowing `java.lang.Package`** – The `Package` type used in the list clashes with the standard Java class.  
   *Recommendation:* Rename the class to something more specific (e.g., `ShippingPackage`).

4. **Null‑safety** – All JSON binding setters call `.equals()` on the incoming string without checking for `null`. This leads to a `NullPointerException` if the JSON payload omits a field.  
   *Recommendation:* Guard against `null` or use `String.equalsIgnoreCase()` with a null‑check.

5. **Validation** – No checks are performed on numeric values. Negative dimensions, weights, or a free‑shipping threshold lower than zero could break business logic downstream.  
   *Recommendation:* Add basic validation in setters or a dedicated `validate()` method.

6. **Thread‑safety** – The mutable state can be corrupted if accessed from multiple threads.  
   *Recommendation:* Either document that the class is single‑threaded or make it immutable (e.g., via a builder pattern).

7. **JSON library choice** – `json-simple` is lightweight but has limited features and is somewhat outdated. Modern projects usually use Jackson or Gson, which can automatically serialise/deserialise enums and handle nulls gracefully.  
   *Recommendation:* Consider migrating to Jackson with custom serializers/deserializers or use `@JsonProperty` annotations.

### 5.2 Edge Cases  
- **Empty `packages` list** – `transformPackage` will still iterate over an empty list, resulting in an empty array; acceptable but might be worth explicitly handling.  
- **Large `orderTotalFreeShipping` or `handlingFees` values** – `BigDecimal` is fine, but no rounding strategy is defined – downstream code must handle this.  
- **`freeShippingEnabled` set to true but `orderTotalFreeShipping` null** – may lead to unexpected behaviour; validation can catch this.

### 5.3 Future Enhancements  
1. **Builder Pattern** – Replace the large set of setters with a fluent builder to enforce immutability and mandatory fields.  
2. **Validation API** – Integrate Java Bean Validation (`javax.validation`) annotations for constraints (e.g., `@Positive`, `@NotNull`).  
3. **Unit Tests** – Add comprehensive tests for JSON round‑trip, edge cases, and validation failures.  
4. **Documentation** – Provide Javadoc on enum usage and explain the significance of each field in shipping calculations.  
5. **Extensibility** – If new shipping options (e.g., “same‑day”) are added, the design should accommodate them without breaking JSON structure.

---

### Bottom‑Line  

`ShippingConfiguration` is a straightforward POJO that captures all the data needed to compute shipping costs and serialise that data to JSON. While functional, its current design suffers from duplicated state, inconsistent naming, and lack of validation. Refactoring to a single source of truth (preferably an immutable object) and adopting a modern JSON library would greatly improve maintainability, robustness, and future extensibility.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.shipping;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import org.json.simple.JSONArray;
import org.json.simple.JSONAware;
import org.json.simple.JSONObject;

/**
 * Object saved in the database maintaining various shipping options
 * @author casams1
 *
 */
public class ShippingConfiguration implements JSONAware {
	
	//enums
	private ShippingType shippingType = ShippingType.NATIONAL;
	private ShippingBasisType shippingBasisType = ShippingBasisType.SHIPPING;
	private ShippingOptionPriceType shippingOptionPriceType = ShippingOptionPriceType.ALL;
	private ShippingPackageType shippingPackageType = ShippingPackageType.ITEM;
	private ShippingDescription shippingDescription = ShippingDescription.SHORT_DESCRIPTION;
	private ShippingType freeShippingType = null;
	
	private int boxWidth = 0;
	private int boxHeight = 0;
	private int boxLength = 0;
	private double boxWeight = 0;
	private double maxWeight = 0;
	
	//free shipping
	private boolean freeShippingEnabled = false;
	private BigDecimal orderTotalFreeShipping = null;
	
	private List<Package> packages = new ArrayList<Package>();

	
	
	private BigDecimal handlingFees = null;
	private boolean taxOnShipping = false;
	
	
	//JSON bindings
	private String shipType;
	private String shipBaseType;
	private String shipOptionPriceType = ShippingOptionPriceType.ALL.name();
	private String shipPackageType;
	private String shipDescription;
	private String shipFreeType;
	
	//Transient
	private String orderTotalFreeShippingText = null;
	private String handlingFeesText = null;
	
	
	public String getShipType() {
		return shipType;
	}


	public String getShipBaseType() {
		return shipBaseType;
	}


	public String getShipOptionPriceType() {
		return shipOptionPriceType;
	}



	public void setShippingOptionPriceType(ShippingOptionPriceType shippingOptionPriceType) {
		this.shippingOptionPriceType = shippingOptionPriceType;
		this.shipOptionPriceType = this.shippingOptionPriceType.name();
	}


	public ShippingOptionPriceType getShippingOptionPriceType() {
		return shippingOptionPriceType;
	}


	public void setShippingBasisType(ShippingBasisType shippingBasisType) {
		this.shippingBasisType = shippingBasisType;
		this.shipBaseType = this.shippingBasisType.name();
	}


	public ShippingBasisType getShippingBasisType() {
		return shippingBasisType;
	}


	public void setShippingType(ShippingType shippingType) {
		this.shippingType = shippingType;
		this.shipType = this.shippingType.name();
	}


	public ShippingType getShippingType() {
		return shippingType;
	}
	
	public ShippingPackageType getShippingPackageType() {
		return shippingPackageType;
	}


	public void setShippingPackageType(ShippingPackageType shippingPackageType) {
		this.shippingPackageType = shippingPackageType;
		this.shipPackageType = shippingPackageType.name();
	}
	
	
	public String getShipPackageType() {
		return shipPackageType;
	}

	
	/** JSON bindding **/
	public void setShipType(String shipType) {
		this.shipType = shipType;
		ShippingType sType = ShippingType.NATIONAL;
		if(shipType.equals(ShippingType.INTERNATIONAL.name())) {
			sType = ShippingType.INTERNATIONAL;
		}
		setShippingType(sType);
	}


	public void setShipOptionPriceType(String shipOptionPriceType) {
		this.shipOptionPriceType = shipOptionPriceType;
		ShippingOptionPriceType sType = ShippingOptionPriceType.ALL;
		if(shipOptionPriceType.equals(ShippingOptionPriceType.HIGHEST.name())) {
			sType = ShippingOptionPriceType.HIGHEST;
		}
		if(shipOptionPriceType.equals(ShippingOptionPriceType.LEAST.name())) {
			sType = ShippingOptionPriceType.LEAST;
		}
		setShippingOptionPriceType(sType);
	}


	public void setShipBaseType(String shipBaseType) {
		this.shipBaseType = shipBaseType;
		ShippingBasisType sType = ShippingBasisType.SHIPPING;
		if(shipBaseType.equals(ShippingBasisType.BILLING.name())) {
			sType = ShippingBasisType.BILLING;
		}
		setShippingBasisType(sType);
	}



	public void setShipPackageType(String shipPackageType) {
		this.shipPackageType = shipPackageType;
		ShippingPackageType sType = ShippingPackageType.ITEM;
		if(shipPackageType.equals(ShippingPackageType.BOX.name())) {
			sType = ShippingPackageType.BOX;
		}
		this.setShippingPackageType(sType);
	}
	
	public void setShipDescription(String shipDescription) {
		this.shipDescription = shipDescription;
		ShippingDescription sType = ShippingDescription.SHORT_DESCRIPTION;
		if(shipDescription.equals(ShippingDescription.LONG_DESCRIPTION.name())) {
			sType = ShippingDescription.LONG_DESCRIPTION;
		}
		this.setShippingDescription(sType);
	}
	
	public void setShipFreeType(String shipFreeType) {
		this.shipFreeType = shipFreeType;
		ShippingType sType = ShippingType.NATIONAL;
		if(shipFreeType.equals(ShippingType.INTERNATIONAL.name())) {
			sType = ShippingType.INTERNATIONAL;
		}
		setFreeShippingType(sType);
	}

	@SuppressWarnings("unchecked")
	@Override
	public String toJSONString() {
		JSONObject data = new JSONObject();
		data.put("shipBaseType", this.getShippingBasisType().name());
		data.put("shipOptionPriceType", this.getShippingOptionPriceType().name());
		data.put("shipType", this.getShippingType().name());
		data.put("shipPackageType", this.getShippingPackageType().name());
		if(shipFreeType!=null) {
			data.put("shipFreeType", this.getFreeShippingType().name());
		}
		data.put("shipDescription", this.getShippingDescription().name());
		
		
		data.put("boxWidth", this.getBoxWidth());
		data.put("boxHeight", this.getBoxHeight());
		data.put("boxLength", this.getBoxLength());
		data.put("boxWeight", this.getBoxWeight());
		data.put("maxWeight", this.getMaxWeight());
		data.put("freeShippingEnabled", this.freeShippingEnabled);
		data.put("orderTotalFreeShipping", this.orderTotalFreeShipping);
		data.put("handlingFees", this.handlingFees);
		data.put("taxOnShipping", this.taxOnShipping);
		
		
		JSONArray jsonArray = new JSONArray();

		for(Package p : this.getPackages()) {
			jsonArray.add(transformPackage(p));
		}
		
		data.put("packages", jsonArray);
		
		
		return data.toJSONString();
	}
	
	@SuppressWarnings("unchecked")
	private JSONObject transformPackage(Package p) {
		JSONObject data = new JSONObject();
		data.put("boxWidth", p.getBoxWidth());
		data.put("boxHeight", p.getBoxHeight());
		data.put("boxLength", p.getBoxLength());
		data.put("boxWeight", p.getBoxWeight());
		data.put("maxWeight", p.getMaxWeight());
		data.put("treshold", p.getTreshold());
		data.put("code", p.getCode());
		data.put("shipPackageType", p.getShipPackageType().name());
		data.put("defaultPackaging", p.isDefaultPackaging());
		return data;
	}


	public int getBoxWidth() {
		return boxWidth;
	}


	public void setBoxWidth(int boxWidth) {
		this.boxWidth = boxWidth;
	}


	public int getBoxHeight() {
		return boxHeight;
	}


	public void setBoxHeight(int boxHeight) {
		this.boxHeight = boxHeight;
	}


	public int getBoxLength() {
		return boxLength;
	}


	public void setBoxLength(int boxLength) {
		this.boxLength = boxLength;
	}


	public double getBoxWeight() {
		return boxWeight;
	}


	public void setBoxWeight(double boxWeight) {
		this.boxWeight = boxWeight;
	}


	public double getMaxWeight() {
		return maxWeight;
	}


	public void setMaxWeight(double maxWeight) {
		this.maxWeight = maxWeight;
	}


	public boolean isFreeShippingEnabled() {
		return freeShippingEnabled;
	}


	public void setFreeShippingEnabled(boolean freeShippingEnabled) {
		this.freeShippingEnabled = freeShippingEnabled;
	}


	public BigDecimal getOrderTotalFreeShipping() {
		return orderTotalFreeShipping;
	}


	public void setOrderTotalFreeShipping(BigDecimal orderTotalFreeShipping) {
		this.orderTotalFreeShipping = orderTotalFreeShipping;
	}


	public void setHandlingFees(BigDecimal handlingFees) {
		this.handlingFees = handlingFees;
	}


	public BigDecimal getHandlingFees() {
		return handlingFees;
	}


	public void setTaxOnShipping(boolean taxOnShipping) {
		this.taxOnShipping = taxOnShipping;
	}


	public boolean isTaxOnShipping() {
		return taxOnShipping;
	}





	public String getShipDescription() {
		return shipDescription;
	}


	public void setShippingDescription(ShippingDescription shippingDescription) {
		this.shippingDescription = shippingDescription;
	}


	public ShippingDescription getShippingDescription() {
		return shippingDescription;
	}


	public void setFreeShippingType(ShippingType freeShippingType) {
		this.freeShippingType = freeShippingType;
	}


	public ShippingType getFreeShippingType() {
		return freeShippingType;
	}



	public String getShipFreeType() {
		return shipFreeType;
	}


	public void setOrderTotalFreeShippingText(String orderTotalFreeShippingText) {
		this.orderTotalFreeShippingText = orderTotalFreeShippingText;
	}


	public String getOrderTotalFreeShippingText() {
		return orderTotalFreeShippingText;
	}


	public void setHandlingFeesText(String handlingFeesText) {
		this.handlingFeesText = handlingFeesText;
	}


	public String getHandlingFeesText() {
		return handlingFeesText;
	}


	public List<Package> getPackages() {
		return packages;
	}


	public void setPackages(List<Package> packages) {
		this.packages = packages;
	}











}





```
