# JWTTokenUtil.java

## Review

## 1. Summary  
`JWTTokenUtil` is a Spring‑managed component that encapsulates all JWT‑related logic for the application.  
It handles:  

* **Token creation** – builds a signed JWT containing username, audience and timestamps.  
* **Claim extraction** – helper methods to pull out subject, issued‑at, expiration, audience, etc.  
* **Validation** – checks username match, expiry, password‑reset constraints and optional “grace” periods.  
* **Refresh** – regenerates a token with a fresh issued‑at/expiration pair.  

The class relies on the **io.jsonwebtoken (JJWT)** library, Spring’s `@Value` injection, and a custom `JWTUser` DTO.  The design follows a fairly common *utility‑service* pattern, keeping JWT logic in one, easily injectable bean.

---

## 2. Detailed Description  

### Core Flow  

| Step | What happens | Key method(s) |
|------|--------------|---------------|
| **Startup** | Spring injects `jwt.secret` and `jwt.expiration` into the bean. | `@Value` annotations |
| **Token Generation** | `generateToken(UserDetails)` is called, which internally calls `doGenerateToken`. | `generateToken`, `doGenerateToken` |
| **Token Validation** | `validateToken(token, userDetails)` is invoked during authentication; it verifies subject, expiry, and password‑reset time. | `validateToken` |
| **Token Refresh** | If a token is eligible, `refreshToken(token)` (or `canTokenBeRefreshed…` helpers) produce a new token with updated timestamps. | `refreshToken`, `canTokenBeRefreshed`, `canTokenBeRefreshedWithGrace` |

### Design Choices & Assumptions  

* **Audience Handling** – The bean assumes that only `api`, `web`, `mobile`, and `tablet` audiences are possible. The *grace period* is applied only to `mobile` and `tablet`.  
* **Expiration Unit** – `expiration` is stored as seconds (Long). It is multiplied by 1000 when computing the expiration date.  
* **Grace Period** – A 200‑second buffer is added to both the *token lifetime* and *password‑reset* checks. This is a hard‑coded value and is only used in the `canTokenBeRefreshedWithGrace` path.  
* **Serialization** – The class implements `Serializable` but never actually serialises state; the interface is superfluous.  
* **Error Handling** – `getAllClaimsFromToken` will throw a runtime exception if the token is malformed or the signature is invalid. No try/catch is used, so callers must handle it.  
* **Logging** – The class uses `System.out.println` for debugging; in production this should be replaced with a proper logger (e.g., SLF4J).  

### Architecture  

The component is *stateless*: all state lives in method parameters or local variables, which makes it thread‑safe. It is a classic *Service* that delegates to the JJWT library. No external framework (apart from Spring) is involved, making it easy to unit‑test by mocking `@Value` fields or by passing in a known secret.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `getUsernameFromToken(String token)` | Return the JWT subject (`sub`). | JWT string | `String` | None |
| `getIssuedAtDateFromToken(String token)` | Return `iat`. | JWT | `Date` | None |
| `getExpirationDateFromToken(String token)` | Return `exp`. | JWT | `Date` | None |
| `getAudienceFromToken(String token)` | Return `aud`. | JWT | `String` | None |
| `getClaimFromToken(String token, Function<Claims,T> resolver)` | Generic claim extractor. | JWT, resolver | `T` | None |
| `getAllClaimsFromToken(String token)` | Parse JWT and return claims body. | JWT | `Claims` | Throws `io.jsonwebtoken.JwtException` if invalid |
| `isTokenExpired(String token)` | Check if `exp` < now. | JWT | `Boolean` | None |
| `isTokenExpiredWithGrace(String token)` | Same as above but add `GRACE_PERIOD`. | JWT | `Boolean` | None |
| `isCreatedBeforeLastPasswordReset(Date created, Date lastPasswordReset)` | Validate `iat` < password reset. | two dates | `Boolean` | None |
| `isCreatedBeforeLastPasswordResetWithGrace(Date created, Date lastPasswordReset)` | Same but with grace. | two dates | `Boolean` | None |
| `addSeconds(Date date, Integer seconds)` | Add seconds to a date. | date, seconds | `Date` | None |
| `generateAudience()` | Hard‑code default audience (`api`). | None | `String` | None |
| `ignoreTokenExpiration(String token)` | Determine if the token’s audience should bypass expiry. | JWT | `Boolean` | None |
| `generateToken(UserDetails userDetails)` | Build a token for a user. | `UserDetails` | `String` | None |
| `doGenerateToken(Map<String,Object> claims, String subject, String audience)` | Low‑level JWT builder. | claims, subject, audience | `String` | None |
| `canTokenBeRefreshedWithGrace(String token, Date lastPasswordReset)` | Determine if a token can be refreshed when grace is considered. | token, lastPasswordReset | `Boolean` | Prints debug info |
| `canTokenBeRefreshed(String token, Date lastPasswordReset)` | Same but without grace. | token, lastPasswordReset | `Boolean` | None |
| `refreshToken(String token)` | Create a new token with same claims but new timestamps. | token | `String` | None |
| `validateToken(String token, UserDetails userDetails)` | Full validation: subject, expiry, password reset. | token, `UserDetails` | `Boolean` | None |
| `calculateExpirationDate(Date createdDate)` | Compute `exp` by adding `expiration` seconds. | date | `Date` | None |

### Reusable / Utility Methods  

* `addSeconds`, `isTokenExpired`, `isTokenExpiredWithGrace`, `isCreatedBeforeLastPasswordReset`, and `isCreatedBeforeLastPasswordResetWithGrace` are pure helper methods that can be reused by other services if the logic is required outside this class.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.beans.factory.annotation.Value` | Spring Core | Injects secret & expiration from `application.properties` |
| `org.springframework.security.core.userdetails.UserDetails` | Spring Security | Interface used for authentication |
| `org.springframework.stereotype.Component` | Spring Core | Marks the class as a bean |
| `com.salesmanager.shop.store.security.user.JWTUser` | Project DTO | Custom `UserDetails` implementation |
| `com.salesmanager.shop.utils.DateUtil` | Project utility | Provides current date (`DateUtil.getDate()`) |
| `io.jsonwebtoken.*` | JJWT library | JWT parsing, building, signing |
| `java.util.*`, `java.util.function.Function` | JDK | Core collections & functional interface |

All external libs are third‑party; only `DateUtil` is project‑specific. There are no platform‑specific constraints beyond running on a JVM with JJWT on the classpath.

---

## 5. Additional Notes & Recommendations  

### 5.1 Bug / Unfinished Code  
`canTokenBeRefreshedWithGrace` currently prints debugging statements and **always returns `true`**.  
```java
return true;
```
This likely defeats the purpose of the grace‑period logic. Either the method should implement the intended logic (similar to the commented out line) or be removed if not needed.

### 5.2 Logging  
Replace all `System.out.println` calls with a proper logger (`org.slf4j.Logger`).  
```java
private static final Logger LOG = LoggerFactory.getLogger(JWTTokenUtil.class);
```

### 5.3 Exception Handling  
`getAllClaimsFromToken` can throw `JwtException`. It might be safer to catch this and return `null` or a custom exception so that callers can react gracefully.

### 5.4 Secret & Expiration Exposure  
The bean stores `secret` and `expiration` as mutable fields (although they are injected once). Consider marking them `final` or providing a constructor‑injected immutable version to make the bean fully immutable.

### 5.5 Constants & Flexibility  
Hard‑coded audience values and the 200‑second grace period limit flexibility. Expose them as configurable properties if the application may need to change them at runtime.

### 5.6 Thread‑Safety & Statelessness  
The component is stateless and thread‑safe as long as `DateUtil.getDate()` is thread‑safe (which it likely is). No synchronization is required.

### 5.7 Unit‑Testing Tips  
* Mock `DateUtil.getDate()` to control “now”.  
* Use a known secret and expiration value to generate tokens and assert that all extraction methods work.  
* Verify that `validateToken` fails when the username mismatches, token is expired, or the password‑reset date is after `iat`.

### 5.8 Possible Enhancements  
1. **Refresh Token Flow** – Add a dedicated refresh token that is longer‑lived and stored server‑side (e.g., in Redis) to avoid re‑authenticating with credentials.  
2. **Role Claims** – Populate the `claims` map with user roles/authorities for downstream authorization checks.  
3. **Token Revocation** – Store a hash of issued tokens or a blacklist to allow immediate revocation.  
4. **Algorithm Flexibility** – Make the signing algorithm configurable (HS256/HS512/RS256, etc.).  
5. **Compact API** – Expose a higher‑level service (`JwtService`) that aggregates common operations (create, validate, refresh) and hides the low‑level JJWT details from the rest of the application.

---

### Bottom‑Line  

`JWTTokenUtil` is a compact, well‑structured utility that covers the essentials of JWT handling in a Spring application. The main areas to polish are:

* Remove the broken `canTokenBeRefreshedWithGrace` logic.  
* Replace debug `println`s with proper logging.  
* Add defensive error handling for malformed tokens.  
* Consider making configurable values externalized for future flexibility.  

Once these improvements are made, the class will be production‑ready, maintainable, and easier to test.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.security;

import java.io.Serializable;
import java.util.Calendar;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import com.salesmanager.shop.store.security.user.JWTUser;
import com.salesmanager.shop.utils.DateUtil;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

/**
 * Used for managing token based authentication for customer and user
 * @author c.samson
 *
 */
@Component
public class JWTTokenUtil implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	
	    static final int GRACE_PERIOD = 200;
	
	
	
	 	static final String CLAIM_KEY_USERNAME = "sub";
	    static final String CLAIM_KEY_AUDIENCE = "aud";
	    static final String CLAIM_KEY_CREATED = "iat";

	    static final String AUDIENCE_UNKNOWN = "unknown";
	    static final String AUDIENCE_API = "api";
	    static final String AUDIENCE_WEB = "web";
	    static final String AUDIENCE_MOBILE = "mobile";
	    static final String AUDIENCE_TABLET = "tablet";


	    @Value("${jwt.secret}")
	    private String secret;

	    @Value("${jwt.expiration}")
	    private Long expiration;

	    public String getUsernameFromToken(String token) {
	        return getClaimFromToken(token, Claims::getSubject);
	    }

	    public Date getIssuedAtDateFromToken(String token) {
	        return getClaimFromToken(token, Claims::getIssuedAt);
	    }

	    public Date getExpirationDateFromToken(String token) {
	        return getClaimFromToken(token, Claims::getExpiration);
	    }

	    public String getAudienceFromToken(String token) {
	        return getClaimFromToken(token, Claims::getAudience);
	    }

	    public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
	        final Claims claims = getAllClaimsFromToken(token);
	        return claimsResolver.apply(claims);
	    }

	    private Claims getAllClaimsFromToken(String token) {
	        return Jwts.parser()
	                .setSigningKey(secret)
	                .parseClaimsJws(token)
	                .getBody();
	    }

	    private Boolean isTokenExpired(String token) {
	        final Date expiration = getExpirationDateFromToken(token);
	        return expiration.before(DateUtil.getDate());
	    }
	    
	    private Boolean isTokenExpiredWithGrace(String token) {
	            Date expiration = getExpirationDateFromToken(token);
	            expiration = addSeconds(expiration,GRACE_PERIOD);
	            return expiration.before(DateUtil.getDate());
	    }

	    private Boolean isCreatedBeforeLastPasswordReset(Date created, Date lastPasswordReset) {
	        return (lastPasswordReset != null && created.before(lastPasswordReset));
	    }
	    
	    private Boolean isCreatedBeforeLastPasswordResetWithGrace(Date created, Date lastPasswordReset) {
	        return (lastPasswordReset != null && created.before(addSeconds(lastPasswordReset,GRACE_PERIOD)));
	    }
	    
	    private Date addSeconds(Date date, Integer seconds) {
	      Calendar cal = Calendar.getInstance();
	      cal.setTime(date);
	      cal.add(Calendar.SECOND, seconds);
	      return cal.getTime();
	    }

	    private String generateAudience() {
	        return AUDIENCE_API;
	    }

	    private Boolean ignoreTokenExpiration(String token) {
	        String audience = getAudienceFromToken(token);
	        return (AUDIENCE_TABLET.equals(audience) || AUDIENCE_MOBILE.equals(audience));
	    }

	    public String generateToken(UserDetails userDetails) {
	        Map<String, Object> claims = new HashMap<>();
	        return doGenerateToken(claims, userDetails.getUsername(), generateAudience());
	    }

	    private String doGenerateToken(Map<String, Object> claims, String subject, String audience) {
	        final Date createdDate = DateUtil.getDate();
	        final Date expirationDate = calculateExpirationDate(createdDate);

	        System.out.println("doGenerateToken " + createdDate);

	        return Jwts.builder()
	                .setClaims(claims)
	                .setSubject(subject)
	                .setAudience(audience)
	                .setIssuedAt(createdDate)
	                .setExpiration(expirationDate)
	                .signWith(SignatureAlgorithm.HS512, secret)
	                .compact();
	    }
	    
        public Boolean canTokenBeRefreshedWithGrace(String token, Date lastPasswordReset) {
          final Date created = getIssuedAtDateFromToken(token);
          boolean t = isCreatedBeforeLastPasswordResetWithGrace(created, lastPasswordReset);
          boolean u = isTokenExpiredWithGrace(token);
          boolean v =  ignoreTokenExpiration(token);
          System.out.println(t + " " +  u + " " + v);
          System.out.println(!isCreatedBeforeLastPasswordResetWithGrace(created, lastPasswordReset)
                  && (!isTokenExpiredWithGrace(token) || ignoreTokenExpiration(token)));
          //return !isCreatedBeforeLastPasswordResetWithGrace(created, lastPasswordReset)
          //        && (!isTokenExpired(token) || ignoreTokenExpiration(token));
          return true;
        }	    

	    public Boolean canTokenBeRefreshed(String token, Date lastPasswordReset) {
	        final Date created = getIssuedAtDateFromToken(token);
	        return !isCreatedBeforeLastPasswordReset(created, lastPasswordReset)
	                && (!isTokenExpired(token) || ignoreTokenExpiration(token));
	    }

	    public String refreshToken(String token) {
	        final Date createdDate = DateUtil.getDate();
	        final Date expirationDate = calculateExpirationDate(createdDate);

	        final Claims claims = getAllClaimsFromToken(token);
	        claims.setIssuedAt(createdDate);
	        claims.setExpiration(expirationDate);

	        return Jwts.builder()
	                .setClaims(claims)
	                .signWith(SignatureAlgorithm.HS512, secret)
	                .compact();
	    }

	    public Boolean validateToken(String token, UserDetails userDetails) {
	        JWTUser user = (JWTUser) userDetails;
	        final String username = getUsernameFromToken(token);
	        final Date created = getIssuedAtDateFromToken(token);
	        //final Date expiration = getExpirationDateFromToken(token);
	        
	        boolean usernameEquals = username.equals(user.getUsername());
	        boolean isTokenExpired = isTokenExpired(token);
	        boolean isTokenCreatedBeforeLastPasswordReset = isCreatedBeforeLastPasswordReset(created, user.getLastPasswordResetDate());
	        
	        return (

	        		usernameEquals && !isTokenExpired && !isTokenCreatedBeforeLastPasswordReset
	        );
	    }

	    private Date calculateExpirationDate(Date createdDate) {
	        return new Date(createdDate.getTime() + expiration * 1000);
	    }

}



```
