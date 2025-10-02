# ZonesLoader.java

## Review

## 1. Summary

**Purpose**  
`ZonesLoader` is a Spring‐managed component that populates `Zone` and `ZoneDescription` objects from JSON files located in `classpath:/reference/zones/*.json`. Each file follows a naming convention of `<country code>_<language code>.json` (e.g., `us_en.json`). The loader supports two modes:

1. **`loadIndividualZones()`** – returns a list of zone maps, one per file, allowing each file to be processed in isolation.
2. **`loadZones(String jsonFilePath)`** – loads a single JSON file into a single map of zones.

**Key Components**

| Component | Role |
|-----------|------|
| `LanguageService` | Retrieves all supported languages. |
| `CountryService` | Retrieves all supported countries. |
| `ResourcePatternResolver` | Scans the classpath for JSON files. |
| `ObjectMapper` (Jackson) | Deserializes JSON into Java `Map`s. |
| `Zone`, `ZoneDescription` | Domain entities that represent a geographic zone and its localized description. |

The loader uses the classic **Data‑Transfer‑Object (DTO)** pattern: raw JSON is parsed into `Map<String,Object>` structures, then transformed into richer domain objects.

**Design Patterns / Frameworks**

* **Spring Component** – dependency injection via `@Component`, `@Inject`, and `@Autowired`.  
* **Repository Pattern** – the `LanguageService` and `CountryService` abstract persistence.  
* **Factory Method** – `mapZone()` encapsulates the complex object construction logic.  
* **Exception Translation** – `ServiceException` wraps low‑level IO or mapping errors.

---

## 2. Detailed Description

### Execution Flow

1. **Initialization** – Spring injects `LanguageService`, `CountryService`, and a `ResourcePatternResolver`.  
2. **File Discovery** – `geZoneFiles(PATH)` uses the resolver to fetch all JSON files under `/reference/zones`.  
3. **Data Loading** – For each file:
   * Read into `InputStream`.
   * Jackson parses into a generic `Map<String,Object>`.  
4. **Language Filtering** – The loader iterates through all languages, matching the file name or the `ALL_REGIONS` key (`"*"`).  
5. **Zone Mapping** – Each zone entry (`zoneCode`, `zoneName`, `countryCode`) is passed to `mapZone()`:
   * A new `Zone` is created if it does not already exist in the map.
   * The country is resolved via `countriesMap`.
   * A `ZoneDescription` is built and added to a per‑zone list.
   * Duplicate detection (`zonesMark`) prevents the same language/zone pair from being processed twice.  
6. **Attachment of Descriptions** – After all entries are processed, each `Zone` receives its list of `ZoneDescription`s.  
7. **Return Value** –  
   * `loadIndividualZones()` returns a `List<Map<String, Zone>>`, one map per file.  
   * `loadZones()` returns a single `Map<String, Zone>` for the requested file.  

### Assumptions & Constraints

* **File Format** – JSON files must contain a top‑level map where each key is either a language code or `"*"` and the value is a list of zone objects.  
* **Case Sensitivity** – File names and language codes are expected to be lower‑case.  
* **Uniqueness** – Zone codes are globally unique across all files; otherwise, later definitions overwrite earlier ones.  
* **Country Lookup** – Every zone must reference a valid country; missing countries result in a logged warning and the zone is skipped.  
* **Resource Availability** – All zone files must be available on the classpath; otherwise `resourceResolver.getResources(PATH)` will throw an `IOException`.  

### Architecture Choices

* **Use of `Map<String,Object>` for JSON** – Avoids a dedicated DTO class, keeping the loader lightweight but sacrificing type safety.  
* **Separate `mapZone()`** – Centralizes object creation, but mixes the responsibilities of parsing, validation, and mapping in one method.  
* **Lazy Loading of Services** – The loader defers country/language lookups until runtime, which is fine for small datasets but could be optimized for large numbers of zones.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `loadIndividualZones()` | Loads all zone files, returning a list of zone maps (one per file). | None | `List<Map<String, Zone>>` | Throws `ServiceException` on failure. |
| `loadZones(String jsonFilePath)` | Loads a single JSON file into a map of zones. | `jsonFilePath` – classpath location of JSON. | `Map<String, Zone>` | Throws `ServiceException` on failure. |
| `mapZone(Language, Map<String, List<ZoneDescription>>, Map<String, Country>, Map<String, Zone>, Map<String, String>, Map<String, String>)` | Constructs/updates `Zone` and `ZoneDescription` instances from a raw map entry. | `Language` – current language. | `void` | Modifies supplied maps; logs warnings. |
| `geZoneFiles(String)` | Retrieves all JSON resources matching the given pattern. | `path` – resource pattern. | `List<Resource>` | Throws `IOException` if resources cannot be located. |
| `loadFileContent(String)` | Unused helper that returns an `InputStream` for a given file name. | `fileName` – file name to load. | `InputStream` | Throws `Exception`. |

**Reusable/Utility Methods**

* `mapZone()` is the main reusable routine for converting a raw zone map into domain objects.  
* `geZoneFiles()` can be reused elsewhere to list zone JSON resources.

---

## 4. Dependencies

| Library | Type | Purpose |
|---------|------|---------|
| `org.slf4j:slf4j-api` | Standard | Logging abstraction. |
| `org.springframework:*` | Framework | Dependency injection, resource resolution, component scanning. |
| `com.fasterxml.jackson.core:jackson-databind` | Third‑party | JSON parsing into generic maps. |
| `com.salesmanager.core.business.services.reference.country.CountryService` | Internal | Fetches countries. |
| `com.salesmanager.core.business.services.reference.language.LanguageService` | Internal | Fetches languages. |
| `com.salesmanager.core.model.reference.*` | Internal | Domain entities (`Country`, `Language`, `Zone`, `ZoneDescription`). |
| `com.salesmanager.core.business.exception.ServiceException` | Internal | Wrapper for checked exceptions. |

No platform‑specific dependencies are required beyond the standard Java SE and Spring ecosystem.

---

## 5. Additional Notes & Recommendations

### Edge Cases & Limitations

1. **Missing or Malformed JSON** – The loader silently skips missing files but throws a `ServiceException` for malformed JSON. No graceful fallback or retry logic.  
2. **Duplicate Zone Codes Across Files** – Later files overwrite earlier ones silently; duplicates are not logged.  
3. **Unmatched Country Codes** – A zone referencing a non‑existent country results in a warning and omission; there is no mechanism to handle incomplete data.  
4. **Resource Path Hard‑Coded** – `PATH` is fixed to `/reference/zones/*.json`; any change requires code modification.  
5. **Large Datasets** – Building `Map` objects for every zone can become memory intensive; no streaming or batch processing.  

### Suggested Enhancements

| Issue | Recommendation |
|-------|----------------|
| **JSON schema validation** | Validate the structure of each JSON file against a schema before mapping to catch errors early. |
| **Use DTOs** | Create `ZoneDTO` classes that map directly from JSON, improving type safety and readability. |
| **Logging improvements** | Use structured logging (e.g., SLF4J markers) to capture zone and file context in warnings. |
| **Configuration** | Externalize the zone path (`zones.path`) into Spring properties to avoid code changes. |
| **Error handling** | Provide more granular exceptions (e.g., `ZoneMappingException`) to allow callers to differentiate issues. |
| **Batch processing** | If the number of zones is large, consider streaming the JSON using Jackson's `ObjectMapper` with `JsonParser` to reduce memory footprint. |
| **Unit tests** | Add tests covering:
   * Valid single and multi‑file loads.
   * Duplicate zone detection.
   * Missing country scenarios. |
| **Documentation** | Expand JavaDoc on `mapZone()` explaining its contract and side effects. |
| **Refactor** | Split responsibilities:
   * `ZoneParser` – handles JSON deserialization.  
   * `ZoneAssembler` – builds domain objects.  
   * `ZoneRepository` – persists zones if needed. |
| **Use of Optional** | Replace `null` checks with `Optional` to avoid potential `NullPointerException`s. |

Overall, `ZonesLoader` serves its purpose but could benefit from clearer separation of concerns, stronger typing, and more robust error handling to improve maintainability and resilience.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.reference.loader;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import javax.inject.Inject;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.ResourcePatternResolver;
import org.springframework.stereotype.Component;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.reference.country.CountryService;
import com.salesmanager.core.business.services.reference.language.LanguageService;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;
import com.salesmanager.core.model.reference.zone.ZoneDescription;

/**
 * Drop files in reference/zones with following format
 * 
 * <country code>_<language code>.json All lower cases
 * 
 * @author carlsamson
 *
 */
@Component
public class ZonesLoader {

	private static final Logger LOGGER = LoggerFactory.getLogger(ZonesLoader.class);

	@Inject
	private LanguageService languageService;

	@Inject
	private CountryService countryService;
	
	@Autowired
	private ResourcePatternResolver resourceResolver;

	private static final String PATH = "classpath:/reference/zones/*.json";

	private static final String ALL_REGIONS = "*";

	//
	@SuppressWarnings({ "rawtypes", "unchecked" })
	public List<Map<String, Zone>> loadIndividualZones() throws Exception {

		List<Map<String, Zone>> loadedZones = new ArrayList<Map<String, Zone>>();
		try {

			List<Resource> files = geZoneFiles(PATH);
			List<Language> languages = languageService.list();

			ObjectMapper mapper = new ObjectMapper();

			List<Country> countries = countryService.list();
			Map<String, Country> countriesMap = new HashMap<String, Country>();
			for (Country country : countries) {
				countriesMap.put(country.getIsoCode(), country);
			}

			Map<String, Zone> zonesMap = new LinkedHashMap<String, Zone>();
			Map<String, List<ZoneDescription>> zonesDescriptionsMap = new LinkedHashMap<String, List<ZoneDescription>>();
			Map<String, String> zonesMark = new LinkedHashMap<String, String>();

			// load files individually
			for (Resource resource : files) {
				InputStream in = resource.getInputStream();
				if(in == null) {
					continue;
				}
				Map<String, Object> data = mapper.readValue(in, Map.class);
				
				if(resource.getFilename().contains("_")) {
					for (Language l : languages) {
						if (resource.getFilename().contains("_" + l.getCode())) {// lead for this
							// language
							List langList = (List) data.get(l.getCode());
							if (langList != null) {
								/**
								 * submethod
								 */
								for (Object z : langList) {
									Map<String, String> e = (Map<String, String>) z;
									mapZone(l, zonesDescriptionsMap, countriesMap, zonesMap, zonesMark, e);
								}
							}
						}
					}
				} else {
					List langList = (List) data.get(ALL_REGIONS);
					if (langList != null) {
						/**
						 * submethod
						 */
						for (Language l : languages) {
							for (Object z : langList) {
								Map<String, String> e = (Map<String, String>) z;
								mapZone(l, zonesDescriptionsMap, countriesMap, zonesMap, zonesMark, e);
							}
						}
					}
				}
				
				for (Map.Entry<String, Zone> entry : zonesMap.entrySet()) {
					String key = entry.getKey();
					Zone value = entry.getValue();

					// get descriptions
					List<ZoneDescription> descriptions = zonesDescriptionsMap.get(key);
					if (descriptions != null) {
						value.setDescriptons(descriptions);
					}
				}

				loadedZones.add(zonesMap);
			}
			return loadedZones;

		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}


	private InputStream loadFileContent(String fileName) throws Exception {
		return this.getClass().getClassLoader().getResourceAsStream("classpath:/reference/zones/" + fileName);
	}

	public Map<String, Zone> loadZones(String jsonFilePath) throws Exception {

		List<Language> languages = languageService.list();

		List<Country> countries = countryService.list();
		Map<String, Country> countriesMap = new HashMap<String, Country>();
		for (Country country : countries) {

			countriesMap.put(country.getIsoCode(), country);

		}

		ObjectMapper mapper = new ObjectMapper();

		try {

			InputStream in = this.getClass().getClassLoader().getResourceAsStream(jsonFilePath);

			@SuppressWarnings("unchecked")
			Map<String, Object> data = mapper.readValue(in, Map.class);

			Map<String, Zone> zonesMap = new HashMap<String, Zone>();
			Map<String, List<ZoneDescription>> zonesDescriptionsMap = new HashMap<String, List<ZoneDescription>>();
			Map<String, String> zonesMark = new HashMap<String, String>();

			for (Language l : languages) {
				@SuppressWarnings("rawtypes")
				List langList = (List) data.get(l.getCode());
				if (langList != null) {
					/**
					 * submethod
					 */
					for (Object z : langList) {
						@SuppressWarnings("unchecked")
						Map<String, String> e = (Map<String, String>) z;
						this.mapZone(l, zonesDescriptionsMap, countriesMap, zonesMap, zonesMark, e);

						/**
						 * String zoneCode = e.get("zoneCode"); ZoneDescription
						 * zoneDescription = new ZoneDescription();
						 * zoneDescription.setLanguage(l);
						 * zoneDescription.setName(e.get("zoneName")); Zone zone
						 * = null; List<ZoneDescription> descriptions = null; if
						 * (!zonesMap.containsKey(zoneCode)) { zone = new
						 * Zone(); Country country =
						 * countriesMap.get(e.get("countryCode")); if (country
						 * == null) { LOGGER.warn("Country is null for " +
						 * zoneCode + " and country code " +
						 * e.get("countryCode")); continue; }
						 * zone.setCountry(country); zonesMap.put(zoneCode,
						 * zone); zone.setCode(zoneCode);
						 * 
						 * }
						 * 
						 * if (zonesMark.containsKey(l.getCode() + "_" +
						 * zoneCode)) { LOGGER.warn("This zone seems to be a
						 * duplicate ! " + zoneCode + " and language code " +
						 * l.getCode()); continue; }
						 * 
						 * zonesMark.put(l.getCode() + "_" + zoneCode,
						 * l.getCode() + "_" + zoneCode);
						 * 
						 * if (zonesDescriptionsMap.containsKey(zoneCode)) {
						 * descriptions = zonesDescriptionsMap.get(zoneCode); }
						 * else { descriptions = new
						 * ArrayList<ZoneDescription>();
						 * zonesDescriptionsMap.put(zoneCode, descriptions); }
						 * 
						 * descriptions.add(zoneDescription);
						 **/

					}
				} 

			}

			for (Map.Entry<String, Zone> entry : zonesMap.entrySet()) {
				String key = entry.getKey();
				Zone value = entry.getValue();

				// get descriptions
				List<ZoneDescription> descriptions = zonesDescriptionsMap.get(key);
				if (descriptions != null) {
					value.setDescriptons(descriptions);
				}
			}

			return zonesMap;

		} catch (Exception e) {
			throw new ServiceException(e);
		}

	}

	// internal complex mapping stuff, don't try this at home ...
	private void mapZone(Language l, Map<String, List<ZoneDescription>> zonesDescriptionsMap,
			Map<String, Country> countriesMap, Map<String, Zone> zonesMap, Map<String, String> zonesMark,
			Map<String, String> list) {

		String zoneCode = list.get("zoneCode");
		ZoneDescription zoneDescription = new ZoneDescription();
		zoneDescription.setLanguage(l);
		zoneDescription.setName(list.get("zoneName"));
		Zone zone = null;
		List<ZoneDescription> descriptions = null;
		if (!zonesMap.containsKey(zoneCode)) {
			zone = new Zone();
			Country country = countriesMap.get(list.get("countryCode"));
			if (country == null) {
				LOGGER.warn("Country is null for " + zoneCode + " and country code " + list.get("countryCode"));
				return;
			}
			zone.setCountry(country);
			zone.setCode(zoneCode);
			zonesMap.put(zoneCode, zone);
			

		}

		if (zonesMark.containsKey(l.getCode() + "_" + zoneCode)) {
			LOGGER.warn("This zone seems to be a duplicate !  " + zoneCode + " and language code " + l.getCode());
			return;
		}

		zonesMark.put(l.getCode() + "_" + zoneCode, l.getCode() + "_" + zoneCode);

		if (zonesDescriptionsMap.containsKey(zoneCode)) {
			descriptions = zonesDescriptionsMap.get(zoneCode);
		} else {
			descriptions = new ArrayList<ZoneDescription>();
			zonesDescriptionsMap.put(zoneCode, descriptions);
		}

		descriptions.add(zoneDescription);

	}

	private List<Resource> geZoneFiles(String path) throws IOException {
		Resource[] resources =resourceResolver.getResources(PATH);

		List<Resource> files = new ArrayList<>();
		Collections.addAll(files, resources);
		return files;

	}





}




```
