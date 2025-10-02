# ShopApplicationConfiguration.java

## Review

## 1. Summary  
**Purpose** – This class is a Spring‑Boot configuration that wires together the web‑layer of the *Shop* module.  
It registers filters, message converters, view controllers, locale handling, and message bundles. It also imports core configuration (`CoreApplicationConfiguration`) and enables Spring Security.  

**Key components**  
| Component | Role |
|-----------|------|
| `XssFilter` | Servlet filter that sanitises user input (applied to `/shop/**`, `/api/**`, `/customer/**`). |
| `CorsFilter` | Handles CORS headers for `/services/**` and `/api/**`. |
| `LocaleChangeInterceptor` + `SessionLocaleResolver` | Switches UI locale via a request parameter. |
| `ByteArrayHttpMessageConverter` | Serialises binary data (images, PDFs, etc.) for responses. |
| `ReloadableResourceBundleMessageSource` | Loads i18n bundles from the `bundles` package. |
| `LabelUtils` | Utility bean for message localisation (likely used by views). |

**Frameworks / Libraries**  
* Spring Boot (MVC, Web, Security)  
* Spring Framework 5.x  
* Apache Commons Logging (used instead of SLF4J)  
* Jackson (via `MappingJackson2HttpMessageConverter`)  

No explicit design patterns beyond Spring’s standard DI and MVC architecture are visible.

---

## 2. Detailed Description  
1. **Initialization** –  
   * The class is annotated with `@Configuration`, `@ComponentScan`, `@ServletComponentScan`, `@Import`, and `@EnableWebSecurity`.  
   * `@EventListener(ApplicationReadyEvent.class)` logs the current working directory when the application starts.

2. **Filters** –  
   * `croseSiteFilter()` (likely a typo, should be *xssSiteFilter*) creates a `FilterRegistrationBean` that registers a fresh `XssFilter` instance.  
   * It applies the filter to three URL patterns.  
   * The filter is not ordered relative to the `CorsFilter`; the default order may place it after CORS processing.

3. **Message Converters** –  
   * `configureMessageConverters` overrides Spring’s default converters and adds only a `MappingJackson2HttpMessageConverter`.  
   * The separately defined `byteArrayHttpMessageConverter()` bean is **not** registered automatically; developers would need to add it manually or override `extendMessageConverters`.  
   * As a result, binary payloads may not be handled correctly unless the bean is wired elsewhere.

4. **View Controllers** –  
   * `/` is mapped directly to the view named `"shop"`.  
   * No view resolver is shown; the view is likely resolved by Thymeleaf or JSP configured elsewhere.

5. **Interceptors** –  
   * `localeChangeInterceptor()` is added globally; it listens for a request param (default `"locale"`) and switches the session locale.  
   * The commented-out `storeFilter` block indicates a potential future filter for store‑specific logic.  
   * `corsFilter()` is added for `/services/**` and `/api/**`.  
   * No ordering is specified, so default precedence applies.

6. **Locale Support** –  
   * `SessionLocaleResolver` sets the default locale to `Locale.getDefault()`.  
   * `ReloadableResourceBundleMessageSource` loads four message bundles from the classpath.

7. **Miscellaneous** –  
   * `LabelUtils` is exposed as a bean; its implementation is not shown, but it likely provides helpers for message formatting.

**Assumptions / Constraints**  
* The application is deployed in a servlet container that supports filters (`FilterRegistrationBean`).  
* JSON payloads are the primary data exchange format (hence the Jackson converter).  
* Binary content handling is required but not wired out‑of‑the‑box.  
* Locale changes are expected to be triggered via request parameters.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `applicationReadyCode()` | Logs the current working directory when the application is ready. | None | None | Logs to application log |
| `croseSiteFilter()` | Creates a `FilterRegistrationBean` for `XssFilter`. | None | `FilterRegistrationBean<XssFilter>` | Registers a servlet filter |
| `configureMessageConverters(List<HttpMessageConverter<?>> converters)` | Adds a Jackson converter to Spring’s list. | `converters` | None | Modifies the provided list |
| `addViewControllers(ViewControllerRegistry registry)` | Maps `/` to the `"shop"` view. | `registry` | None | Adds view mapping |
| `addInterceptors(InterceptorRegistry registry)` | Registers locale and CORS interceptors. | `registry` | None | Adds interceptors |
| `byteArrayHttpMessageConverter()` | Builds a `ByteArrayHttpMessageConverter` that supports images and generic binary streams. | None | `ByteArrayHttpMessageConverter` | Creates a converter bean |
| `localeChangeInterceptor()` | Returns a `LocaleChangeInterceptor`. | None | `LocaleChangeInterceptor` | Creates interceptor bean |
| `corsFilter()` | Returns a `CorsFilter` bean. | None | `CorsFilter` | Creates filter bean |
| `localeResolver()` | Provides a session‑based locale resolver. | None | `SessionLocaleResolver` | Creates resolver bean |
| `messageSource()` | Configures an i18n message source with several base names. | None | `ReloadableResourceBundleMessageSource` | Creates message source bean |
| `messages()` | Exposes a `LabelUtils` instance. | None | `LabelUtils` | Creates utility bean |

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `org.springframework.boot:spring-boot-starter-web` | Third‑party | Provides MVC, Jackson, servlet API, etc. |
| `org.springframework.boot:spring-boot-starter-security` | Third‑party | Enables `@EnableWebSecurity`. |
| `org.apache.commons:commons-logging` | Third‑party | Logging façade (not SLF4J). |
| `com.fasterxml.jackson.core:jackson-databind` | Third‑party | Implicit via Spring Boot. |
| `com.salesmanager.core` | Third‑party | `CoreApplicationConfiguration` (external module). |
| `com.salesmanager.shop.filter.CorsFilter` / `XssFilter` | Internal | Custom filters. |
| `com.salesmanager.shop.utils.LabelUtils` | Internal | Utility for localisation. |

No platform‑specific dependencies are visible; everything is standard for a Spring Boot web application.

---

## 5. Additional Notes  

### Strengths  
* Centralised configuration for locale, messages, filters, and converters.  
* Clear separation of concerns (security, web, core).  
* Uses Spring’s bean lifecycle to inject custom components.

### Potential Issues / Edge Cases  
1. **`croseSiteFilter` typo** – May cause confusion or hinder IDE refactoring.  
2. **Filter ordering** – No explicit order; if both `XssFilter` and `CorsFilter` need to execute in a specific sequence, use `setOrder()` on `FilterRegistrationBean`.  
3. **Missing binary converter registration** – The `byteArrayHttpMessageConverter` bean is never added to the list of converters, so binary responses may fail. Override `extendMessageConverters` or register it explicitly.  
4. **LocaleChangeInterceptor without parameter name** – Default param is `"locale"`; if the UI uses a different name, requests won’t trigger a locale change.  
5. **`ReloadableResourceBundleMessageSource` missing fallback** – If a key is missing in one bundle, the next is not checked unless `fallbackToSystemLocale` is true.  
6. **Logging with Commons Logging** – Modern projects usually use SLF4J. Mixing logging APIs may result in duplicate logs or missing messages.  
7. **`@ServletComponentScan`** – If no `@WebFilter`/`@WebListener` annotations exist elsewhere, this annotation is unnecessary.  

### Future Enhancements  
* **Order the filters**: `FilterRegistrationBean#setOrder()` for deterministic processing.  
* **Add CORS configuration**: Use Spring’s `CorsConfigurationSource` bean instead of a custom filter if you need fine‑grained control.  
* **Extend message converters**: Ensure binary converters are part of the conversion pipeline.  
* **Internationalisation**: Expose a `LocaleResolver` that also falls back to a default locale when a session value is missing.  
* **Testing**: Add unit tests for the configuration (e.g., using `@WebMvcTest` to verify view mapping, locale changes, and filter registration).  
* **Documentation**: Add Javadoc to public beans, especially `XssFilter` and `CorsFilter`, so downstream developers know their usage.  

Overall, the configuration is concise and follows Spring Boot conventions, but a few small adjustments would improve robustness and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.application.config;

import static org.springframework.http.MediaType.APPLICATION_OCTET_STREAM;
import static org.springframework.http.MediaType.IMAGE_GIF;
import static org.springframework.http.MediaType.IMAGE_JPEG;
import static org.springframework.http.MediaType.IMAGE_PNG;

import java.util.Arrays;
import java.util.List;
import java.util.Locale;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletComponentScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.context.event.EventListener;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;
import org.springframework.http.MediaType;
import org.springframework.http.converter.ByteArrayHttpMessageConverter;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;

import com.salesmanager.core.business.configuration.CoreApplicationConfiguration;
import com.salesmanager.shop.filter.CorsFilter;
import com.salesmanager.shop.filter.XssFilter;
import com.salesmanager.shop.utils.LabelUtils;

@Configuration
@ComponentScan({"com.salesmanager.shop"})
@ServletComponentScan
@Import({CoreApplicationConfiguration.class}) // import sm-core configurations
@EnableWebSecurity
public class ShopApplicationConfiguration implements WebMvcConfigurer {

  protected final Log logger = LogFactory.getLog(getClass());

  @EventListener(ApplicationReadyEvent.class)
  public void applicationReadyCode() {
    String workingDir = System.getProperty("user.dir");
    logger.info("Current working directory : " + workingDir);
  }

  @Bean
  public FilterRegistrationBean<XssFilter> croseSiteFilter(){
      FilterRegistrationBean<XssFilter> registrationBean 
        = new FilterRegistrationBean<>();
          
      registrationBean.setFilter(new XssFilter());
      registrationBean.addUrlPatterns("/shop/**");
      registrationBean.addUrlPatterns("/api/**");
      registrationBean.addUrlPatterns("/customer/**");
          
      return registrationBean;    
  }

  @Override
  public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(new MappingJackson2HttpMessageConverter());
  }

  @Override
  public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("shop");
  }

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    // Changes the locale when a 'locale' request parameter is sent; e.g. /?locale=de
    registry.addInterceptor(localeChangeInterceptor());

    /**
    registry
        .addInterceptor(storeFilter())
        // store web front filter
        .addPathPatterns("/shop/**")
        // customer section filter
        .addPathPatterns("/customer/**");
     **/

    registry
        .addInterceptor(corsFilter())
        // public services cors filter
        .addPathPatterns("/services/**")
        // REST api
        .addPathPatterns("/api/**");

  }

  @Bean
  public ByteArrayHttpMessageConverter byteArrayHttpMessageConverter() {
    List<MediaType> supportedMediaTypes = Arrays.asList(IMAGE_JPEG, IMAGE_GIF, IMAGE_PNG, APPLICATION_OCTET_STREAM);

    ByteArrayHttpMessageConverter byteArrayHttpMessageConverter =
        new ByteArrayHttpMessageConverter();
    byteArrayHttpMessageConverter.setSupportedMediaTypes(supportedMediaTypes);
    return byteArrayHttpMessageConverter;
  }

  @Bean
  public LocaleChangeInterceptor localeChangeInterceptor() {
    return new LocaleChangeInterceptor();
  }

	/*
	 * @Bean public StoreFilter storeFilter() { return new StoreFilter(); }
	 */

  @Bean
  public CorsFilter corsFilter() {
    return new CorsFilter();
  }


  @Bean
  public SessionLocaleResolver localeResolver() {
    SessionLocaleResolver slr = new SessionLocaleResolver();
    slr.setDefaultLocale(Locale.getDefault());
    return slr;
  }

  @Bean
  public ReloadableResourceBundleMessageSource messageSource() {
    ReloadableResourceBundleMessageSource messageSource =
        new ReloadableResourceBundleMessageSource();
    messageSource.setBasenames(
        "classpath:bundles/shopizer",
        "classpath:bundles/messages",
        "classpath:bundles/shipping",
        "classpath:bundles/payment");

    messageSource.setDefaultEncoding("UTF-8");
    return messageSource;
  }

  @Bean
  public LabelUtils messages() {
    return new LabelUtils();
  }

}



```
