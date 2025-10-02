# SecurityFacadeImpl.java

## Review

## 1. Summary  

`SecurityFacadeImpl` is a Spring‑managed service that implements the `SecurityFacade` interface.  
Its responsibilities are:

| Responsibility | Implementation |
|----------------|----------------|
| **Password validation** – ensures a plain‑text password satisfies a regex rule. |
| **Password encoding / matching** – uses Spring Security’s `PasswordEncoder`. |
| **Permission retrieval** – translates a list of group names into a list of readable permissions. |

The class pulls in three collaborators:

* `PermissionService` – fetches permissions from the persistence layer.  
* `GroupService` – resolves group names to `Group` entities.  
* `PasswordEncoder` – encodes/matches passwords.

The code is annotated with Spring’s `@Service`, and the collaborators are injected with JSR‑330’s `@Inject`. The only external library beyond Spring is **jsoup’s** `Validate` helper.

---

## 2. Detailed Description  

### Flow of Execution

1. **`validateUserPassword(String)`**  
   * Uses a pre‑compiled regex (`USER_PASSWORD_PATTERN`) to check that the password contains at least one lower‑case letter, one upper‑case letter, one digit, and is 6–12 characters long.

2. **`encodePassword(String)`**  
   * Delegates to `PasswordEncoder.encode`.

3. **`matchPassword(String, String)`**  
   * Delegates to `PasswordEncoder.matches`.  
   * **Bug** – the order of the arguments is reversed (`newPassword` is supplied as the raw password and `modelPassword` as the encoded one).  

4. **`matchRawPasswords(String, String)`**  
   * Uses jsoup’s `Validate.notNull` to guard against nulls and then compares the two strings with `equals`.

5. **`getPermissions(List<String>)`**  
   * Resolves group names to `Group` entities via `groupService.listGroupByNames`.  
   * Builds a set of group IDs and passes them to `PermissionService.listByCriteria`.  
   * **Current state** – throws `ServiceRuntimeException("Not implemented")` and finally returns `null`.  The method never actually returns a permission list.  
   * Errors from the service layer are caught, stack‑traces printed, and the method silently returns `null`.  

### Dependencies and Constraints

| Dependency | Purpose | Standard/Third‑party |
|------------|---------|----------------------|
| `PasswordEncoder` (Spring Security) | Encode and match passwords | Third‑party |
| `PermissionService`, `GroupService` | Domain‑specific services | Third‑party (SalesManager core) |
| `org.jsoup.helper.Validate` | Null‑check utility | Third‑party (jsoup) |
| `Pattern`, `Matcher` | Regex for password validation | JDK |
| `@Service`, `@Inject` | Spring bean lifecycle | Spring, JSR‑330 |

Assumptions made by the implementation:

* The `groups` list is non‑null; otherwise `listGroupByNames` may throw an exception.
* The `PermissionService` and `GroupService` are thread‑safe.
* The caller will not rely on the return value of `getPermissions` until it is fully implemented.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects | Notes |
|--------|---------|--------|---------|--------------|-------|
| `getPermissions(List<String> groups)` | Retrieve readable permissions for the supplied group names. | `List<String> groups` | `List<ReadablePermission>` (currently `null`) | None except exception handling | **Not implemented** – throws `ServiceRuntimeException`. Should return a populated list or throw a more specific exception. |
| `validateUserPassword(String password)` | Verify that a plain‑text password satisfies policy. | `String password` | `boolean` (true if regex matches) | None | Uses a compiled regex (`USER_PASSWORD_PATTERN`). |
| `encodePassword(String password)` | Encode a plain‑text password. | `String password` | `String` (encoded hash) | None | Delegates to `PasswordEncoder.encode`. |
| `matchPassword(String modelPassword, String newPassword)` | Compare raw password against an encoded one. | `String modelPassword` (encoded), `String newPassword` (raw) | `boolean` | None | **Bug** – argument order reversed. |
| `matchRawPasswords(String password, String repeatPassword)` | Simple equality check after null guard. | `String password`, `String repeatPassword` | `boolean` | None | Uses jsoup’s `Validate.notNull`. |

**Reusable/Utility Methods**  
None of the methods are marked as private helpers; the class is essentially a thin façade.

---

## 4. Dependencies  

| Library | Role | Notes |
|---------|------|-------|
| **Spring Framework** (`@Service`, `@Inject`, `PasswordEncoder`) | DI, security | Standard in a Spring application. |
| **Spring Security** (`PasswordEncoder`) | Password hashing | Requires a bean definition (e.g., `BCryptPasswordEncoder`). |
| **SalesManager Core** (`PermissionService`, `GroupService`, `Group`, `PermissionCriteria`, `PermissionList`) | Domain services | Must be wired into the application context. |
| **jsoup** (`Validate`) | Null validation | Could be replaced with `org.apache.commons.lang3.Validate` or Java 8 `Objects.requireNonNull`. |
| **Java SE** (`Pattern`, `Matcher`, collections) | Core functionality | No external jars. |

No platform‑specific assumptions beyond a standard Spring application context.

---

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns – password handling is isolated from permission retrieval.  
* Use of a compiled regex improves performance.  
* Leveraging Spring Security’s `PasswordEncoder` ensures industry‑grade hashing.

### Weaknesses / Risks  
1. **Unimplemented `getPermissions`** – throws an exception and returns `null`.  This will break any consumer expecting a list.  
2. **Argument order bug in `matchPassword`** – could lead to authentication failures or security holes.  
3. **Exception handling** – `ServiceException` is caught, stack trace printed, and method returns `null`.  This swallows errors and makes debugging harder.  Prefer re‑throwing as a runtime exception or returning an empty list.  
4. **Null handling** – `getPermissions` does not guard against a `null` input list; this may trigger a `NullPointerException`.  
5. **Use of jsoup Validate** – while functional, jsoup is not a typical dependency for validation; consider a lighter library or plain Java checks.  
6. **Thread‑safety** – `userPasswordPattern` is immutable, so safe. Other components depend on their own thread‑safety guarantees.  

### Edge Cases  
* Empty `groups` list → `groupService.listGroupByNames` may return an empty list; code will proceed but ultimately return `null`.  
* `null` password → `validateUserPassword` will throw `NullPointerException` due to `matcher` call.  
* Extremely long passwords may exceed stack size during regex evaluation (unlikely with 6–12 char rule).

### Future Enhancements  
* Implement `getPermissions` fully: convert `PermissionList` into `ReadablePermission` list and return it.  
* Fix the argument order in `matchPassword`.  
* Replace `Validate.notNull` with Java 8 `Objects.requireNonNull`.  
* Centralise error handling: create a custom runtime exception for permission lookup failures.  
* Add unit tests covering password policy, encoding, matching, and permission retrieval.  
* Consider caching group IDs for frequent lookups if performance becomes an issue.  
* Document the password policy and expected exception types in Javadoc.

Overall, the skeleton is useful but requires completion and bug fixes before it can be used safely in production.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.security.facade;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import javax.inject.Inject;

import org.jsoup.helper.Validate;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.user.GroupService;
import com.salesmanager.core.business.services.user.PermissionService;
import com.salesmanager.core.model.user.Group;
import com.salesmanager.core.model.user.PermissionCriteria;
import com.salesmanager.core.model.user.PermissionList;
import com.salesmanager.shop.model.security.ReadablePermission;
import com.salesmanager.shop.store.api.exception.ServiceRuntimeException;

@Service("securityFacade")
public class SecurityFacadeImpl implements SecurityFacade {
  
  private static final String USER_PASSWORD_PATTERN = "((?=.*[a-z])(?=.*\\d)(?=.*[A-Z]).{6,12})";
  
  private Pattern userPasswordPattern = Pattern.compile(USER_PASSWORD_PATTERN);

  @Inject
  private PermissionService permissionService;

  @Inject
  private GroupService groupService;
  
  @Inject
  private PasswordEncoder passwordEncoder;

  @SuppressWarnings({"rawtypes", "unchecked"})
  @Override
  public List<ReadablePermission> getPermissions(List<String> groups) {

    List<Group> userGroups = null;
    try {
      userGroups = groupService.listGroupByNames(groups);

      List<Integer> ids = new ArrayList<Integer>();
      for (Group g : userGroups) {
        ids.add(g.getId());
      }

      PermissionCriteria criteria = new PermissionCriteria();
      criteria.setGroupIds(new HashSet(ids));

      PermissionList permissions = permissionService.listByCriteria(criteria);
      throw new ServiceRuntimeException("Not implemented");
    } catch (ServiceException e) {
      e.printStackTrace();
    }

    return null;
  }

  @Override
  public boolean validateUserPassword(String password) {

    Matcher matcher = userPasswordPattern.matcher(password);
    return matcher.matches();
  }

  @Override
  public String encodePassword(String password) {
    return passwordEncoder.encode(password);
  }

  /**
   * Match non encoded to encoded
   * Don't use this as a simple raw password check
   */
  @Override
  public boolean matchPassword(String modelPassword, String newPassword) {
    return passwordEncoder.matches(newPassword, modelPassword);
  }

@Override
public boolean matchRawPasswords(String password, String repeatPassword) {
	Validate.notNull(password,"password is null");
	Validate.notNull(repeatPassword,"repeat password is null");
	return password.equals(repeatPassword);
}
  
  

}



```
