# DroolsBeanFactory.java

## Review

## 1. Summary  
`DroolsBeanFactory` is a Spring‐managed component that builds and provides Drools rule containers (`KieContainer`) and sessions (`KieSession`).  
- **Purpose**: Dynamically load DRL/Excel rules from the classpath, build a KIE module, and expose sessions for the application to fire rules.  
- **Key components**:  
  - `KieServices` – the Drools API entry point.  
  - `KieFileSystem` – holds rule files before they are built.  
  - `KieBuilder` – compiles the files into a `KieModule`.  
  - `KieContainer` – the runtime container for sessions.  
- **Notable patterns**: The factory pattern (singletons provided by Spring) and the Builder pattern (KIE API). No heavy frameworks beyond Spring and Drools.

---

## 2. Detailed Description  

### Flow of execution  

| Phase | What happens | Comments |
|-------|--------------|----------|
| **Injection** | `priceByDistance` and `shippingDecision` are injected from `application.properties`. | The property values must be class‑path resource names (e.g. `rules/priceByDistance.drl`). |
| **KIE Setup** | `getKieContainer()` calls `getKieRepository()` (adds the default module), builds all files in the KieFileSystem, then creates a `KieContainer` from the module’s ReleaseId. | `getKieRepository()` is redundant – the default repository is added automatically. |
| **Session creation** | `getKieSession()` repeats the same steps as `getKieContainer()` but finally returns a fresh session. | Duplicate code – consider extracting a shared method. |
| **Single‑resource session** | `getKieSession(Resource dt)` builds a container from a single rule resource and returns a session. | Useful for ad‑hoc rules but also repeats logic. |
| **Debug helper** | `getDrlFromExcel()` converts an Excel decision table to DRL text. | Good for debugging but should not be part of the production factory. |

### Assumptions & Constraints  

| Item | Assumption | Impact |
|------|------------|--------|
| **Classpath location** | Rules are located on the classpath and paths are correct. | If a file is missing, `ResourceFactory.newClassPathResource()` silently returns a resource that fails only during build. |
| **Build success** | No validation of `KieBuilder` results. | Build failures silently propagate; rules may never fire. |
| **Thread‑safety** | Each call builds a fresh container/session. | Works but is expensive for high‑frequency usage. |
| **Singleton vs. prototype** | Spring will treat `DroolsBeanFactory` as a singleton. | All session factories share the same `KieServices` instance (fine). |

### Architecture  

The code follows a *factory* design: callers obtain a `KieSession` from this single component. It leverages the *Builder* pattern inherent in Drools (`KieBuilder`). However, the implementation is somewhat manual: it constructs a new `KieFileSystem` and rebuilds the module on every request. A more efficient design would create a **single** `KieContainer` at startup (or when rules change) and re‑use it, only re‑building if a rule file changes.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getKieFileSystem()` | Builds a `KieFileSystem` and writes the two rule files defined in properties. | None | `KieFileSystem` | Throws `IOException` if a rule resource cannot be read. |
| `getKieContainer()` | Returns a fully built `KieContainer`. | None | `KieContainer` | Re‑builds the module on each call. |
| `getKieRepository()` | Adds the default release to the repository. | None | None | No observable effect beyond what KieServices already does. |
| `getKieSession()` | Creates a new `KieSession` from the two rule files. | None | `KieSession` | Re‑builds the module each call. |
| `getKieSession(Resource dt)` | Creates a session from a single rule resource. | `Resource dt` | `KieSession` | Re‑builds only that rule. |
| `getDrlFromExcel(String excelFile)` | Converts an Excel decision table to DRL text (debug helper). | `String excelFile` | `String` (DRL) | No side‑effects. |

### Reusable/Utility Methods  

- `getKieFileSystem()` is the main reusable piece; the other methods could delegate to it.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.drools:drools-core` (and transitive `drools-compiler`, `drools-decisiontables`) | Third‑party | Provides KIE API, decision tables, and compilation. |
| `org.springframework:spring-context` | Third‑party | For `@Component` and `@Value`. |
| `org.springframework` annotations (`@Component`, `@Value`) | Standard | Part of Spring. |
| `java.util` | Standard | Lists, arrays. |
| `java.io` | Standard | IOException handling. |

No platform‑specific dependencies; the code runs on any JVM that has the classpath resources.

---

## 5. Additional Notes & Recommendations  

### Strengths  

* Clear separation of concerns – rule loading, building, and session creation are encapsulated.  
* Uses Spring annotations for configuration, enabling external property injection.  
* Provides a handy debug method to inspect Excel decision tables.

### Weaknesses & Edge Cases  

1. **Redundant & Repetitive Code**  
   * `getKieContainer()`, `getKieSession()`, and `getKieSession(Resource)` share almost identical logic.  
   * Refactor into a single private helper that accepts the `KieFileSystem` or `List<Resource>`.

2. **Inefficient Re‑builds**  
   * Every call rebuilds the module, which can be costly in production.  
   * Prefer a singleton `KieContainer` that is rebuilt only when a rule file changes (watcher or versioning).

3. **No Build‑Error Handling**  
   * `KieBuilder.buildAll()` returns a `Results` object.  
   * Add a check for `results.hasMessages(Message.Level.ERROR)` and throw a runtime exception or log.

4. **Unnecessary Repository Manipulation**  
   * `getKieRepository()` merely adds the default release – KieServices already provides this.  
   * Remove it unless a specific use case requires custom ReleaseIds.

5. **Hard‑coded Path Prefix**  
   * `RULES_PATH` is used only in `getKieSession()`; `getKieFileSystem()` expects full paths in the properties.  
   * Decide on a single strategy: either all properties are full paths or all are relative to `RULES_PATH`.

6. **Thread Safety & Caching**  
   * The factory is a singleton, but each session is built from scratch.  
   * Consider adding `@Scope("prototype")` for sessions or returning a pooled container.

7. **Exception Handling**  
   * Methods declare `throws IOException` but no callers handle it; wrap in unchecked exceptions or provide a graceful fallback.

8. **Logging**  
   * Add SLF4J logging to trace build progress, errors, and which rules are loaded.

9. **Unit Tests**  
   * Tests should verify that rules are correctly compiled, that build errors surface, and that sessions can fire rules.

### Future Enhancements  

* **Hot‑Reload** – Monitor the classpath for changes and rebuild the container automatically.  
* **External Configuration** – Store rule locations and ReleaseIds in a YAML file or database, allowing runtime changes.  
* **Performance Monitoring** – Time the build process and expose metrics via Micrometer.  
* **Factory Bean** – Expose a `@Bean` of type `KieContainer` so that other components can inject it directly.  

---

### Final Verdict  

`DroolsBeanFactory` provides the necessary plumbing to load Drools rules in a Spring application, but it suffers from duplicated code, lack of error handling, and inefficiencies. Refactoring to a single, cached `KieContainer`, improving error reporting, and simplifying the repository interaction will make the component more robust, maintainable, and performant.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.configuration;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

import org.drools.decisiontable.DecisionTableProviderImpl;
import org.kie.api.KieServices;
import org.kie.api.builder.KieBuilder;
import org.kie.api.builder.KieFileSystem;
import org.kie.api.builder.KieModule;
import org.kie.api.builder.KieRepository;
import org.kie.api.builder.ReleaseId;
import org.kie.api.io.Resource;
import org.kie.api.runtime.KieContainer;
import org.kie.api.runtime.KieSession;
import org.kie.internal.builder.DecisionTableConfiguration;
import org.kie.internal.builder.DecisionTableInputType;
import org.kie.internal.builder.KnowledgeBuilderFactory;
import org.kie.internal.io.ResourceFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class DroolsBeanFactory {
	
	
	@Value("${config.shipping.rule.priceByDistance}")
	private String priceByDistance;
	
	@Value("${config.shipping.rule.shippingModuleDecision}")
	private String shippingDecision;
	

    private static final String RULES_PATH = "com/salesmanager/drools/rules/";
    private KieServices kieServices = KieServices.Factory.get();

    private  KieFileSystem getKieFileSystem() throws IOException{
        KieFileSystem kieFileSystem = kieServices.newKieFileSystem();
        List<String> rules=Arrays.asList(priceByDistance,shippingDecision);
        for(String rule:rules){
            kieFileSystem.write(ResourceFactory.newClassPathResource(rule));
        }
        return kieFileSystem;
    }

    public KieContainer getKieContainer() throws IOException {
        getKieRepository();

        KieBuilder kb = kieServices.newKieBuilder(getKieFileSystem());
        kb.buildAll();

        KieModule kieModule = kb.getKieModule();

        return kieServices.newKieContainer(kieModule.getReleaseId());

    }

    private void getKieRepository() {
        final KieRepository kieRepository = kieServices.getRepository();
        kieRepository.addKieModule(kieRepository::getDefaultReleaseId);
    }

    public KieSession getKieSession(){
        getKieRepository();
        KieFileSystem kieFileSystem = kieServices.newKieFileSystem();

        kieFileSystem.write(ResourceFactory.newClassPathResource(RULES_PATH + priceByDistance));
        kieFileSystem.write(ResourceFactory.newClassPathResource(RULES_PATH + shippingDecision));
        
        KieBuilder kb = kieServices.newKieBuilder(kieFileSystem);
        kb.buildAll();
        KieModule kieModule = kb.getKieModule();

        KieContainer kContainer = kieServices.newKieContainer(kieModule.getReleaseId());

        return kContainer.newKieSession();

    }

    public KieSession getKieSession(Resource dt) {
        KieFileSystem kieFileSystem = kieServices.newKieFileSystem()
            .write(dt);

       KieBuilder kieBuilder = kieServices.newKieBuilder(kieFileSystem)
            .buildAll();

        KieRepository kieRepository = kieServices.getRepository();

        ReleaseId krDefaultReleaseId = kieRepository.getDefaultReleaseId();

        KieContainer kieContainer = kieServices.newKieContainer(krDefaultReleaseId);

        return kieContainer.newKieSession();
    }

    /*
     * Can be used for debugging
     * Input excelFile example: com/baeldung/drools/rules/Discount.xls
     */
    public String getDrlFromExcel(String excelFile) {
        DecisionTableConfiguration configuration = KnowledgeBuilderFactory.newDecisionTableConfiguration();
        configuration.setInputType(DecisionTableInputType.XLS);

        Resource dt = ResourceFactory.newClassPathResource(excelFile, getClass());

        DecisionTableProviderImpl decisionTableProvider = new DecisionTableProviderImpl();

        return decisionTableProvider.loadFromResource(dt, null);
    }

}



```
