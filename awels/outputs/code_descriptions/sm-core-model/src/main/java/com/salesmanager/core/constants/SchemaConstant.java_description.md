# SchemaConstant.java

## Review

## 1. Summary  

**Purpose**  
`SchemaConstant` is a static configuration holder that defines a collection of read‑only constants used throughout the *SalesManager* core module. Its responsibilities are:

| Category | What it provides | Typical consumers |
|----------|------------------|-------------------|
| Schema | `SALESMANAGER_SCHEMA` | Database schema names |
| Languages | `LANGUAGE_ISO_CODE` | Localization support, default language list |
| Description IDs | `DESCRIPTION_ID_*` | Generation of reference data IDs |
| Regions/Countries | `ALL_REGIONS`, `COUNTRY_ISO_CODE` | Validation, filtering of geographic data |
| Locale lookup | `LOCALES` | Mapping of country ISO code → `Locale` object |
| Currency | `CURRENCY_MAP` | Human‑readable names for ISO currency codes |

**Key Components**  
* Immutable `public static final` primitives and arrays for configuration values.  
* Two static initialisers that populate `LOCALES` and `CURRENCY_MAP`.  
* A deprecated language array with a comment explaining why it’s kept for backwards compatibility.

**Notable Patterns**  
* Singleton‑style static constants (no instance creation).  
* Lazy‑static initialisation via a static block.  
* Use of Java standard library (`java.util.Locale`, `HashMap`).  
* No external frameworks – purely core JDK.

---

## 2. Detailed Description  

### 2.1 Execution Flow  

1. **Class Loading** – When the class is first referenced, the JVM loads it and executes the static fields and blocks in the order they appear.  
2. **Constant Declaration** – All `public static final` primitives and arrays are initialised immediately.  
3. **LOCALES Map** – The first static block iterates over `Locale.getAvailableLocales()` and stores a mapping of *country code → Locale*. If a country code appears multiple times, the last entry wins (the map is overwritten).  
4. **CURRENCY_MAP** – The second static block manually populates a `HashMap` with ISO‑4217 codes and human‑readable currency names. Several duplicate keys exist; the later insertions overwrite earlier ones.  
5. **Class is Ready** – After the static initialisers finish, the class is fully initialised and its constants are accessible anywhere in the application.

### 2.2 Design Choices  

* **Immutability** – All constants are declared `final`, ensuring compile‑time safety.  
* **Global Access** – The class is deliberately *public* and contains no instance methods, making it convenient to reference from any package.  
* **Explicit Overwrites** – The duplicated keys in `CURRENCY_MAP` are a by‑product of copy‑and‑paste; they may hide data‑inconsistency bugs.  
* **Deprecated Array** – `LANGUAGE_ISO_CODE` is marked `@Deprecated`. The comment indicates that only English, French, and Spanish descriptions will be used for reference data.  
* **Locale Cache** – Building `LOCALES` once avoids repeated expensive lookups but consumes memory proportional to the number of locales.

---

## 3. Functions/Methods  

| Method | Description | Parameters | Return | Side Effects |
|--------|-------------|------------|--------|--------------|
| `static` block (first) | Populates `LOCALES` with country → `Locale` mapping. | None | None | Modifies static `LOCALES` |
| `static` block (second) | Populates `CURRENCY_MAP` with currency code → human‑readable name. | None | None | Modifies static `CURRENCY_MAP` |

> **Note** – The class has no public methods; only static initialisers are executed during class load.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `java.util.HashMap` | Standard JDK | Used for two lookup tables. |
| `java.util.Locale` | Standard JDK | Provides available locales. |
| `java.util.Locale#getAvailableLocales` | Standard JDK | Source of country codes. |

No third‑party libraries, frameworks, or external APIs are involved.

---

## 5. Additional Notes  

### 5.1 Duplicate Map Keys  
`CURRENCY_MAP` contains repeated keys (e.g., `"EUR"`, `"USD"`, `"AUD"`, `"XOF"`, `"CURRENCY_MAP.put("GBP", ...)` appears three times). The final value wins silently, making earlier entries unreachable. This can:

* Hide data entry mistakes.
* Reduce the map’s reliability for lookup by currency code.

**Recommendation** – Clean up the initialiser, keep a single entry per code, and add a defensive check (e.g., throw an exception if a duplicate key is detected).

### 5.2 Deprecated Language Codes  
The array is annotated `@Deprecated`, yet it is still part of the public API. If the intention is to remove it in a future release, consider:

* Adding a clear deprecation message in the Javadoc.
* Providing a migration helper (e.g., `getSupportedLanguages()` that returns the current array).

### 5.3 Locale Map Overwrites  
The current logic will overwrite entries for countries that appear in multiple locales (e.g., `Locale("en", "US")` and `Locale("en", "US")`). If the intention is to keep the *first* locale per country, change the insertion logic to guard against overwrites:

```java
LOCALES.putIfAbsent(locale.getCountry(), locale);
```

### 5.4 Thread‑Safety  
`HashMap` is not thread‑safe, but the map is populated once during class loading (which is thread‑safe). After that, it is effectively immutable. If you plan to mutate it later, consider `Collections.unmodifiableMap` or `ConcurrentHashMap`.

### 5.5 Memory Footprint  
`LOCALES` and `CURRENCY_MAP` are static and live for the JVM lifetime. In typical usage this is fine, but if the application is highly memory‑constrained or supports dynamic configuration reloads, a different strategy (e.g., lazy loading or using `Map.ofEntries`) may be preferable.

### 5.6 Internationalization Improvement  
Instead of a hard‑coded `LANGUAGE_ISO_CODE` array, a `Set<String>` derived from `Locale.getISOLanguages()` filtered by your business rules would automatically stay in sync with Java’s locale data.

### 5.7 Future Enhancements  

| Idea | Benefit | Suggested Implementation |
|------|---------|--------------------------|
| **Enum for countries** | Type safety & validation | Create `CountryCode` enum with ISO field, fallback to `LOCALES` for mapping. |
| **Currency name lookup via `Currency.getInstance()`** | Avoid manual map & reduce duplication | Use `Currency.getInstance(code).getDisplayName()` with a locale. |
| **Dynamic loading from properties** | Easier to change without recompilation | Load `languages`, `countries`, `currencies` from a properties file. |
| **Unit tests** | Ensure map contents and deprecation | JUnit tests asserting expected size, presence, and no duplicates. |

---

### 5.8 Summary of Issues & Fixes  

| Issue | Impact | Fix |
|-------|--------|-----|
| Duplicate currency keys | Silent data loss | Remove duplicates, enforce uniqueness. |
| Deprecated language array still exported | Confusing API | Add Javadoc deprecation message; eventually remove or hide. |
| Overwrites in LOCALES map | Potential wrong locale selection | Use `putIfAbsent` or store `List<Locale>` per country. |
| Manual currency map | Hard‑to‑maintain, large | Consider using `Currency` class or external file. |
| Large memory usage | Minor but noticeable | Consider `Collections.unmodifiableMap` or `Map.ofEntries`. |

---

**Conclusion**  
`SchemaConstant` is a straightforward constants holder, but it contains a few maintenance pitfalls (duplicate keys, deprecated fields, potential data loss). Addressing these will improve the reliability, clarity, and future‑proofing of the module.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.constants;

import java.util.HashMap;
import java.util.Locale;

public class SchemaConstant {

	public final static String SALESMANAGER_SCHEMA = "SALESMANAGER";

	/**
	 * Languages iso codes
	 * 
	 */
	//public static final String[] LANGUAGE_ISO_CODE = {"en", "fr", "ru", "es", "ar"};

	@Deprecated
	public static final String[] LANGUAGE_ISO_CODE = {"en", "fr", "es"}; //which descriptions will ve used for reference data
	
	public final static int DESCRIPTION_ID_ALLOCATION_SIZE = 1;
	public final static int DESCRIPTION_ID_START_VALUE = 2000;
	
	/**
	 * All regions
	 */
	public static final String ALL_REGIONS = "*";

	/**
	 * Country iso codes
	 */
	public static final String[] COUNTRY_ISO_CODE = { "AF","AX","AL","DZ",
		"AS","AD","AO","AI","AQ","AG","AR","AM","AW","AU","AT","AZ","BS","BH",
		"BD","BB","BY","BE","BZ","BJ","BM","BT","BO","BA","BW","BV","BR","IO",
		"BN","BG","BF","BI","KH","CM","CA","CV","KY","CF","TD","CL","CN","CX",
		"CC","CO","KM","CG","CD","CK","CR","CI","HR","CU","CY","CZ","DK","DJ",
		"DM","DO","EC","EG","SV","GQ","ER","EE","ET","FK","FO","FJ","FI","FR",
		"GF","PF","TF","GA","GM","GE","DE","GH","GI","GR","GL","GD","GP","GU",
		"GT","GG","GN","GW","GY","HT","HM","VA","HN","HK","HU","IS","IN","ID",
		"IR","IQ","IE","IM","IL","IT","JM","JP","JE","JO","KZ","KE","KI","KP",
		"KR","KW","KG","LA","LV","LB","LS","LR","LY","LI","LT","LU","MO","MK",
		"MG","MW","MY","MV","ML","MT","MH","MQ","MR","MU","YT","MX","FM","MD",
		"MC","MN","ME","MS","MA","MZ","MM","NA","NR","NP","NL","AN","NC","NZ",
		"NI","NE","NG","NU","NF","MP","NO","OM","PK","PW","PS","PA","PG","PY",
		"PE","PH","PN","PL","PT","PR","QA","RE","RO","RU","RW","SH","KN","LC",
		"PM","VC","WS","SM","ST","SA","SN","RS","SC","SL","SG","SK","SI","SB",
		"SO","ZA","GS","ES","LK","SD","SR","SJ","SZ","SE","CH","SY","TW","TJ",
		"TZ","TH","TL","TG","TK","TO","TT","TN","TR","TM","TC","TV","UG","UA",
		"AE","GB","US","UM","UY","UZ","VU","VE","VN","VG","VI","WF","EH",
	    "YE","ZM","ZW" };

	/**
	 * Locale per country iso codes
	 */
	public static final HashMap<String, Locale> LOCALES = new HashMap<String, Locale>();

	static {
		for (Locale locale : Locale.getAvailableLocales()) {
			LOCALES.put(locale.getCountry(), locale);
		}
	}
	
	/**
	 * Currency codes with name
	 */
	public static final HashMap<String, String> CURRENCY_MAP = new HashMap<String, String>();
	
	static {
		CURRENCY_MAP.put("AFN", "Afghani");
		CURRENCY_MAP.put("EUR", "Euro");
		CURRENCY_MAP.put("ALL", "Lek");
		CURRENCY_MAP.put("DZD", "Algerian Dinar");
		CURRENCY_MAP.put("USD", "US Dollar");
		CURRENCY_MAP.put("AOA", "Kwanza");
		CURRENCY_MAP.put("XCD", "East Caribbean Dollar");
		CURRENCY_MAP.put("ARS", "Argentine Peso");
		CURRENCY_MAP.put("AMD", "Armenian Dram");
		CURRENCY_MAP.put("AWG", "Aruban Florin");
		CURRENCY_MAP.put("AUD", "Australian Dollar");
		CURRENCY_MAP.put("AZN", "Azerbaijanian Manat");
		CURRENCY_MAP.put("BSD", "Bahamian Dollar");
		CURRENCY_MAP.put("BHD", "Bahraini Dinar");
		CURRENCY_MAP.put("BDT", "Taka");
		CURRENCY_MAP.put("BBD", "Barbados Dollar");
		CURRENCY_MAP.put("BYR", "Belarussian Ruble");
		CURRENCY_MAP.put("BZD", "Belize Dollar");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("BMD", "Bermudian Dollar");
		CURRENCY_MAP.put("BTN", "Ngultrum");
		CURRENCY_MAP.put("INR", "Indian Rupee");
		CURRENCY_MAP.put("BOB", "Boliviano");
		CURRENCY_MAP.put("BOV", "Mvdol");
		CURRENCY_MAP.put("BAM", "Convertible Mark");
		CURRENCY_MAP.put("BWP", "Pula");
		CURRENCY_MAP.put("NOK", "Norwegian Krone");
		CURRENCY_MAP.put("BRL", "Brazilian Real");
		CURRENCY_MAP.put("BND", "Brunei Dollar");
		CURRENCY_MAP.put("BGN", "Bulgarian Lev");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("BIF", "Burundi Franc");
		CURRENCY_MAP.put("KHR", "Riel");
		CURRENCY_MAP.put("XAF", "CFA Franc BEAC");
		CURRENCY_MAP.put("CAD", "Canadian Dollar");
		CURRENCY_MAP.put("CVE", "Cape Verde Escudo");
		CURRENCY_MAP.put("KYD", "Cayman Islands Dollar");
		CURRENCY_MAP.put("CLF", "Unidades de fomento");
		CURRENCY_MAP.put("CLP", "Chilean Peso");
		CURRENCY_MAP.put("CNY", "Yuan Renminbi");
		CURRENCY_MAP.put("COP", "Colombian Peso");
		//CURRENCY_MAP.put("COU", "Unidad de Valor Real");
		CURRENCY_MAP.put("KMF", "Comoro Franc");
		CURRENCY_MAP.put("XAF", "CFA Franc BEAC");
		CURRENCY_MAP.put("CDF", "Congolese Franc");
		CURRENCY_MAP.put("NZD", "New Zealand Dollar");
		CURRENCY_MAP.put("CRC", "Costa Rican Colon");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("HRK", "Croatian Kuna");
		//CURRENCY_MAP.put("CUC", "Peso Convertible");
		CURRENCY_MAP.put("CUP", "Cuban Peso");
		CURRENCY_MAP.put("ANG", "Netherlands Antillean Guilder");
		CURRENCY_MAP.put("CZK", "Czech Koruna");
		CURRENCY_MAP.put("DKK", "Danish Krone");
		CURRENCY_MAP.put("DJF", "Djibouti Franc");
		CURRENCY_MAP.put("XCD", "East Caribbean Dollar");
		CURRENCY_MAP.put("DOP", "Dominican Peso");
		CURRENCY_MAP.put("EGP", "Egyptian Pound");
		CURRENCY_MAP.put("SVC", "El Salvador Colon");
		CURRENCY_MAP.put("XAF", "CFA Franc BEAC");
		CURRENCY_MAP.put("ERN", "Nakfa");
		CURRENCY_MAP.put("ETB", "Ethiopian Birr");
		CURRENCY_MAP.put("FKP", "Falkland Islands Pound");
		CURRENCY_MAP.put("DKK", "Danish Krone");
		CURRENCY_MAP.put("FJD", "Fiji Dollar");
		CURRENCY_MAP.put("XPF", "CFP Franc");
		CURRENCY_MAP.put("XAF", "CFA Franc BEAC");
		CURRENCY_MAP.put("GMD", "Dalasi");
		CURRENCY_MAP.put("GEL", "Lari");
		CURRENCY_MAP.put("GHS", "Ghana Cedi");
		CURRENCY_MAP.put("GIP", "Gibraltar Pound");
		CURRENCY_MAP.put("DKK", "Danish Krone");
		CURRENCY_MAP.put("XCD", "East Caribbean Dollar");
		CURRENCY_MAP.put("GTQ", "Quetzal");
		CURRENCY_MAP.put("GBP", "Pound Sterling");
		CURRENCY_MAP.put("GNF", "Guinea Franc");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("GYD", "Guyana Dollar");
		CURRENCY_MAP.put("HTG", "Gourde");
		CURRENCY_MAP.put("HNL", "Lempira");
		CURRENCY_MAP.put("HKD", "Hong Kong Dollar");
		CURRENCY_MAP.put("HUF", "Forint");
		CURRENCY_MAP.put("ISK", "Iceland Krona");
		CURRENCY_MAP.put("INR", "Indian Rupee");
		CURRENCY_MAP.put("IDR", "Rupiah");
		CURRENCY_MAP.put("XDR", "SDR (Special Drawing Right)");
		CURRENCY_MAP.put("IRR", "Iranian Rial");
		CURRENCY_MAP.put("IQD", "Iraqi Dinar");
		CURRENCY_MAP.put("GBP", "Pound Sterling");
		CURRENCY_MAP.put("ILS", "New Israeli Sheqel");
		CURRENCY_MAP.put("JMD", "Jamaican Dollar");
		CURRENCY_MAP.put("JPY", "Yen");
		CURRENCY_MAP.put("GBP", "Pound Sterling");
		CURRENCY_MAP.put("JOD", "Jordanian Dinar");
		CURRENCY_MAP.put("KZT", "Tenge");
		CURRENCY_MAP.put("KES", "Kenyan Shilling");
		CURRENCY_MAP.put("AUD", "Australian Dollar");
		CURRENCY_MAP.put("KPW", "North Korean Won");
		CURRENCY_MAP.put("KRW", "Won");
		CURRENCY_MAP.put("KWD", "Kuwaiti Dinar");
		CURRENCY_MAP.put("KGS", "Som");
		CURRENCY_MAP.put("LAK", "Kip");
		CURRENCY_MAP.put("LVL", "Latvian Lats");
		CURRENCY_MAP.put("LBP", "Lebanese Pound");
		CURRENCY_MAP.put("LSL", "Loti");
		CURRENCY_MAP.put("ZAR", "Rand");
		CURRENCY_MAP.put("LRD", "Liberian Dollar");
		CURRENCY_MAP.put("LYD", "Libyan Dinar");
		CURRENCY_MAP.put("CHF", "Swiss Franc");
		CURRENCY_MAP.put("LTL", "Lithuanian Litas");
		CURRENCY_MAP.put("EUR", "Euro");
		CURRENCY_MAP.put("MOP", "Pataca");
		CURRENCY_MAP.put("MKD", "Denar");
		CURRENCY_MAP.put("MGA", "Malagasy Ariary");
		CURRENCY_MAP.put("MWK", "Kwacha");
		CURRENCY_MAP.put("MYR", "Malaysian Ringgit");
		CURRENCY_MAP.put("MVR", "Rufiyaa");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("MRO", "Ouguiya");
		CURRENCY_MAP.put("MUR", "Mauritius Rupee");
		//CURRENCY_MAP.put("XUA", "ADB Unit of Account");
		CURRENCY_MAP.put("MXN", "Mexican Peso");
		CURRENCY_MAP.put("MXV", "Mexican Unidad de Inversion (UDI)");
		CURRENCY_MAP.put("MDL", "Moldovan Leu");
		CURRENCY_MAP.put("MNT", "Tugrik");
		CURRENCY_MAP.put("XCD", "East Caribbean Dollar");
		CURRENCY_MAP.put("MAD", "Moroccan Dirham");
		CURRENCY_MAP.put("MZN", "Mozambique Metical");
		CURRENCY_MAP.put("MMK", "Kyat");
		CURRENCY_MAP.put("NAD", "Namibia Dollar");
		CURRENCY_MAP.put("ZAR", "Rand");
		CURRENCY_MAP.put("NPR", "Nepalese Rupee");
		CURRENCY_MAP.put("XPF", "CFP Franc");
		CURRENCY_MAP.put("NZD", "New Zealand Dollar");
		CURRENCY_MAP.put("NIO", "Cordoba Oro");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("NGN", "Naira");
		CURRENCY_MAP.put("NZD", "New Zealand Dollar");
		CURRENCY_MAP.put("AUD", "Australian Dollar");
		CURRENCY_MAP.put("NOK", "Norwegian Krone");
		CURRENCY_MAP.put("OMR", "Rial Omani");
		CURRENCY_MAP.put("PKR", "Pakistan Rupee");
		CURRENCY_MAP.put("PAB", "Balboa");
		CURRENCY_MAP.put("PGK", "Kina");
		CURRENCY_MAP.put("PYG", "Guarani");
		CURRENCY_MAP.put("PEN", "Nuevo Sol");
		CURRENCY_MAP.put("PHP", "Philippine Peso");
		CURRENCY_MAP.put("NZD", "New Zealand Dollar");
		CURRENCY_MAP.put("PLN", "Zloty");
		CURRENCY_MAP.put("USD", "US Dollar");
		CURRENCY_MAP.put("QAR", "Qatari Rial");
		CURRENCY_MAP.put("RON", "New Romanian Leu");
		CURRENCY_MAP.put("RUB", "Russian Ruble");
		CURRENCY_MAP.put("RWF", "Rwanda Franc");
		CURRENCY_MAP.put("SHP", "Saint Helena Pound");
		CURRENCY_MAP.put("XCD", "East Caribbean Dollar");
		CURRENCY_MAP.put("WST", "Tala");
		CURRENCY_MAP.put("STD", "Dobra");
		CURRENCY_MAP.put("SAR", "Saudi Riyal");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("RSD", "Serbian Dinar");
		CURRENCY_MAP.put("SCR", "Seychelles Rupee");
		CURRENCY_MAP.put("SLL", "Leone");
		CURRENCY_MAP.put("SGD", "Singapore Dollar");
		CURRENCY_MAP.put("ANG", "Netherlands Antillean Guilder");
		//CURRENCY_MAP.put("XSU", "Sucre");
		CURRENCY_MAP.put("SBD", "Solomon Islands Dollar");
		CURRENCY_MAP.put("SOS", "Somali Shilling");
		CURRENCY_MAP.put("ZAR", "Rand");
		//CURRENCY_MAP.put("SSP", "South Sudanese Pound");
		CURRENCY_MAP.put("LKR", "Sri Lanka Rupee");
		CURRENCY_MAP.put("SDG", "Sudanese Pound");
		CURRENCY_MAP.put("SRD", "Surinam Dollar");
		CURRENCY_MAP.put("NOK", "Norwegian Krone");
		CURRENCY_MAP.put("SZL", "Lilangeni");
		CURRENCY_MAP.put("SEK", "Swedish Krona");
		//CURRENCY_MAP.put("CHE", "WIR Euro");
		CURRENCY_MAP.put("CHF", "Swiss Franc");
		//CURRENCY_MAP.put("CHW", "WIR Franc");
		CURRENCY_MAP.put("SYP", "Syrian Pound");
		CURRENCY_MAP.put("TWD", "New Taiwan Dollar");
		CURRENCY_MAP.put("TJS", "Somoni");
		CURRENCY_MAP.put("TZS", "Tanzanian Shilling");
		CURRENCY_MAP.put("THB", "Baht");
		CURRENCY_MAP.put("USD", "US Dollar");
		CURRENCY_MAP.put("XOF", "CFA Franc BCEAO");
		CURRENCY_MAP.put("NZD", "New Zealand Dollar");
		CURRENCY_MAP.put("TTD", "Trinidad and Tobago Dollar");
		CURRENCY_MAP.put("TND", "Tunisian Dinar");
		CURRENCY_MAP.put("TRY", "Turkish Lira");
		//CURRENCY_MAP.put("TMT", "Turkmenistan New Manat");
		CURRENCY_MAP.put("AUD", "Australian Dollar");
		CURRENCY_MAP.put("UGX", "Uganda Shilling");
		CURRENCY_MAP.put("UAH", "Hryvnia");
		CURRENCY_MAP.put("AED", "UAE Dirham");
		CURRENCY_MAP.put("GBP", "Pound Sterling");
		//CURRENCY_MAP.put("UYI", "Uruguay Peso en Unidades Indexadas (URUIURUI)");
		CURRENCY_MAP.put("UYU", "Peso Uruguayo");
		CURRENCY_MAP.put("UZS", "Uzbekistan Sum");
		CURRENCY_MAP.put("VUV", "Vatu");
		CURRENCY_MAP.put("VEF", "Bolivar Fuerte");
		CURRENCY_MAP.put("VND", "Dong");
		CURRENCY_MAP.put("USD", "US Dollar");
		CURRENCY_MAP.put("XPF", "CFP Franc");
		CURRENCY_MAP.put("MAD", "Moroccan Dirham");
		CURRENCY_MAP.put("YER", "Yemeni Rial");
		CURRENCY_MAP.put("ZMK", "Zambian Kwacha");
		//CURRENCY_MAP.put("ZWL", "Zimbabwe Dollar");
		


	}
}



```
