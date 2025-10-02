# UserFacade.java

## Review

## 1. Summary

**Purpose**  
`UserFacade` is a Java interface that defines the contract for user‑management operations in a multi‑store e‑commerce platform. It encapsulates common CRUD, authentication, authorization, and password‑reset logic that would be implemented by concrete service classes.

**Key Components**  
| Component | Role |
|-----------|------|
| `findByUserName`, `findById` | Retrieval of user details (read‑only) |
| `create`, `update`, `delete` | Lifecycle management of `PersistableUser` objects |
| `listByCriteria`, `getByCriteria` | Pagination & filtering of user lists |
| `findPermissionsByGroups` | Permission resolution for user groups |
| `authorizedStore`, `authorizeStore`, `authorizedGroup`, `userInRoles`, `authorizedGroups` | Authorization checks (store‑level, group‑level, role‑level) |
| `sendResetPasswordEmail`, `requestPasswordReset`, `verifyPasswordRequestToken`, `resetPassword` | Password‑reset workflow |
| `changePassword`, `updateEnabled` | Account‑maintenance functions |
| `authenticatedUser` | Helper for retrieving the current logged‑in username |

**Notable Design Patterns / Libraries**  
* **Facade Pattern** – The interface acts as a façade over a potentially complex user service stack.  
* **DTOs** – Uses `PersistableUser`, `ReadableUser`, `ReadableUserList`, `ReadablePermission`, `UserPassword` as data transfer objects.  
* **Criteria Pattern** – `Criteria` / `UserCriteria` objects encapsulate filtering, sorting and pagination.  
* **Spring Security / Role‑Based Access** – Methods such as `authorizedStore` hint at integration with a security framework (likely Spring Security).  

---

## 2. Detailed Description

### Core Flow

1. **Initialization** – A concrete implementation would be injected (via Spring `@Service` or similar) wherever the facade is needed (controllers, argument resolvers, etc.).  
2. **Runtime** –  
   * **Read** – `findByUserName`, `findById` and the two *list* methods fetch user data from the persistence layer, returning read‑only DTOs.  
   * **Create / Update** – `create` / `update` persist `PersistableUser` objects, typically after validation.  
   * **Delete** – `delete` removes a user by ID and store.  
   * **Authorization** – Several methods (e.g., `authorizedStore`, `authorizedGroup`) query user roles/groups to decide if an action is allowed.  
   * **Password** – The reset flow consists of `requestPasswordReset` → `verifyPasswordRequestToken` → `resetPassword`. `sendResetPasswordEmail` is used for the initial email.  
3. **Cleanup** – No explicit resource cleanup; underlying implementation may manage transactions, connections, etc.  

### Assumptions & Constraints

| Assumption | Impact |
|------------|--------|
| `MerchantStore` uniquely identifies a store context | Methods require store information to isolate data per merchant. |
| `Language` is passed for localization of read DTOs | DTOs are language‑aware; the implementation must fetch localized strings. |
| Users belong to groups/roles that map to permissions | Authorization logic expects a group → permission mapping. |
| Email sending is handled externally | `sendResetPasswordEmail` likely delegates to an email service. |
| The interface is deprecated in one method (`getByCriteria`) | Indicates a shift from `Criteria` to `UserCriteria`. |

### Architecture & Design Choices

* **Separation of Concerns** – By using a facade, the presentation layer is decoupled from business logic.  
* **Read vs Persistable DTOs** – Clear distinction between data that can be mutated and data that is immutable to the caller.  
* **Method Overloading** – Multiple `findByUserName` signatures provide convenience for callers that may or may not need store/language context.  
* **Deprecated Method** – Signals evolving API; clients should migrate to `listByCriteria`.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `findByUserName(String, String, Language)` | Retrieve user by username in a specific store & language | `userName`, `storeCode`, `lang` | `ReadableUser` | None |
| `findByUserName(String)` | Retrieve user by username globally | `userName` | `ReadableUser` | None |
| `findById(Long, MerchantStore, Language)` | Retrieve user by ID within a store & language | `id`, `store`, `lang` | `ReadableUser` | None |
| `create(PersistableUser, MerchantStore)` | Persist a new user | `user`, `store` | `ReadableUser` | Creates DB record |
| `findPermissionsByGroups(List<Integer>)` | Resolve permissions for group IDs | `ids` | `List<ReadablePermission>` | None |
| `authorizedStore(String, String)` | Check if user can act on a store | `userName`, `merchantStoreCode` | `boolean` | None |
| `authorizeStore(MerchantStore, String)` | Argument‑resolver helper | `store`, `path` | `boolean` | None |
| `authorizedGroup(String, List<String>)` | Verify user belongs to any of given groups | `userName`, `groupNames` | None (throws if unauthorized) | Potential exception |
| `userInRoles(String, List<String>)` | Check if user has any of the roles | `userName`, `groupNames` | `boolean` | None |
| `sendResetPasswordEmail(ReadableUser, MerchantStore, Language)` | Email reset link | `user`, `store`, `lang` | None | Sends email |
| `authenticatedUser()` | Retrieve username of current auth context | None | `String` | None |
| `getByCriteria(Language, String, Criteria)` *(Deprecated)* | Legacy list retrieval | `language`, `draw`, `criteria` | `ReadableUserList` | None |
| `listByCriteria(UserCriteria, int, int, Language)` | Paginated, filtered list | `criteria`, `page`, `count`, `lang` | `ReadableUserList` | None |
| `delete(Long, String)` | Remove user by ID & store | `id`, `storeCode` | None | Deletes DB record |
| `update(Long, String, MerchantStore, PersistableUser)` | Update user details | `id`, `authenticatedUser`, `store`, `user` | `ReadableUser` | Updates DB |
| `changePassword(Long, String, UserPassword)` | Change password (authenticated user) | `userId`, `authenticatedUser`, `changePassword` | None | Updates password hash |
| `authorizedGroups(String, PersistableUser)` | Verify group assignments on update | `authenticatedUser`, `user` | None | May throw |
| `updateEnabled(MerchantStore, PersistableUser)` | Toggle user enabled flag | `store`, `user` | None | Updates flag |
| `requestPasswordReset(String, String, MerchantStore, Language)` | Initiate reset flow | `userName`, `userContextPath`, `store`, `lang` | None | Sends email, creates token |
| `verifyPasswordRequestToken(String, String)` | Validate token for store | `token`, `store` | None | Throws if invalid |
| `resetPassword(String, String, String)` | Set new password using token | `password`, `token`, `store` | None | Persists new password |

### Reusable / Utility Methods
* `authorizedStore`, `authorizeStore`, `authorizedGroup`, `userInRoles`, `authorizedGroups` – common authorization helpers that can be reused across controllers or services.  
* `authenticatedUser` – convenient shortcut for retrieving the current principal’s username.

---

## 4. Dependencies

| External Library | Usage | Standard / Third‑Party |
|------------------|-------|------------------------|
| `com.salesmanager.core` | Core domain models (`Criteria`, `MerchantStore`, `Language`, `UserCriteria`) | Third‑party (SalesManager core) |
| `com.salesmanager.shop.model` | DTOs (`ReadablePermission`, `PersistableUser`, `ReadableUser`, `ReadableUserList`, `UserPassword`) | Third‑party (SalesManager shop layer) |
| `java.util` | Collections (`List`) | Standard |
| *Implicit* | Spring (likely) for DI, Security context | Third‑party |
| *Implicit* | Email service, token generation | Third‑party |

The interface itself does not import any platform‑specific APIs, but concrete implementations will likely depend on Spring, JPA/Hibernate, and an email/notification system.

---

## 5. Additional Notes

### Strengths
* **Clear separation** between read‑only and mutable DTOs.  
* **Explicit store & language parameters** promote multi‑tenant and localization support.  
* **Comprehensive set of methods** covering CRUD, authorization, and password workflows.  
* **Deprecated method** signals API evolution and encourages migration to newer patterns.

### Potential Issues / Edge Cases
1. **Over‑loading of `findByUserName`** – Might lead to ambiguity in dependency injection contexts or method resolution.  
2. **Missing return value for `authorizedGroup`** – It returns `void`, which may throw an exception on failure. Documenting the exception contract is essential.  
3. **Token handling** – `verifyPasswordRequestToken` and `resetPassword` accept raw token strings; security considerations (e.g., token leakage, expiration handling) must be enforced in the implementation.  
4. **Error handling** – Methods like `delete`, `update`, and `changePassword` return `void` but may throw unchecked exceptions; callers need to know the failure modes.  
5. **Deprecated `getByCriteria`** – If legacy clients still use it, ensure backward compatibility until a clean migration path is documented.  
6. **Language in password reset** – `requestPasswordReset` receives a `Language` but does not expose a `Locale` or region; locale‑specific email templates must be managed carefully.  

### Future Enhancements
* **Unified Exception Hierarchy** – Define a `UserFacadeException` hierarchy to standardize error handling.  
* **Async Operations** – Methods like `sendResetPasswordEmail` could be made asynchronous to improve responsiveness.  
* **Pagination Abstraction** – Replace raw `page` and `count` ints with a `PageRequest` DTO for consistency.  
* **DTO Validation** – Add validation annotations (e.g., `@NotNull`, `@Email`) to DTOs to catch invalid input early.  
* **Security Annotations** – Consider using Spring Security method security (`@PreAuthorize`) at the implementation level to enforce constraints declaratively.  
* **Audit Trail** – Expose hooks for logging changes (create, update, delete) for compliance.  

Overall, `UserFacade` offers a solid foundation for user management in a multi‑store e‑commerce platform. Ensuring consistent implementation of the methods, clear contract documentation, and robust error handling will be key to its reliability and maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.controller.user.facade;

import java.util.List;

import com.salesmanager.core.model.common.Criteria;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.user.UserCriteria;
import com.salesmanager.shop.model.security.ReadablePermission;
import com.salesmanager.shop.model.user.PersistableUser;
import com.salesmanager.shop.model.user.ReadableUser;
import com.salesmanager.shop.model.user.ReadableUserList;
import com.salesmanager.shop.model.user.UserPassword;

/**
 * Access to all methods for managing users
 * 
 * @author carlsamson
 *
 */
public interface UserFacade {

  /**
   * Finds a User by userName
   * 
   * @return
   * @throws Exception
   */
  ReadableUser findByUserName(String userName, String storeCode, Language lang);
  
  /**
   * Find user by userName
   * @param userName
   * @return
   */
  ReadableUser findByUserName(String userName);
  
  /**
   * Find user by id
   * @param id
   * @param store
   * @param lang
   * @return
   */
  ReadableUser findById(Long id, MerchantStore store, Language lang);

  /**
   * Creates a User
   * @param user
   * @param store
   */
  ReadableUser create(PersistableUser user, MerchantStore store);

  /**
   * List permissions by group
   * 
   * @param ids
   * @return
   * @throws Exception
   */
  List<ReadablePermission> findPermissionsByGroups(List<Integer> ids);

  /**
   * Determines if a user is authorized to perform an action on a specific store
   * 
   * @param userName
   * @param merchantStoreCode
   * @return
   * @throws Exception
   */
  boolean authorizedStore(String userName, String merchantStoreCode);
  
  
  /**
   * Method to be used in argument resolver.
   * @param store
   * @return
   */
  boolean authorizeStore(MerchantStore store, String path);
  
  /**
   * Determines if a user is in a specific group
   * @param userName
   * @param groupNames
   */
  void authorizedGroup(String userName, List<String> groupNames);
  
  /**
   * Check if user is in specific list of roles
   * @param userName
   * @param groupNames
   * @return
   */
  boolean userInRoles(String userName, List<String> groupNames);
  
  
  /**
   * Sends reset password email
   * @param user
   * @param store
   * @param language
   */
  void sendResetPasswordEmail(ReadableUser user, MerchantStore store, Language language);
  
  /**
   * Retrieve authenticated user
   * @return
   */
  String authenticatedUser();
  
  /**
   * Get by criteria
   * @param criteria
   * @return
   */
  @Deprecated
  ReadableUserList getByCriteria(Language language,String draw,Criteria criteria);
  
  /**
   * List users
   * @param criteria
   * @param page
   * @param count
   * @param language
   * @return
   */
  ReadableUserList listByCriteria (UserCriteria criteria, int page, int count, Language language);
  
  /**
   * Delete user
   * @param id
   */
  void delete(Long id, String storeCode);
  
  /**
   * Update User
   * @param user
   */
  ReadableUser update(Long id, String authenticatedUser, MerchantStore store, PersistableUser user);
  
  /**
   * Change password request
   * @param userId
   * @param authenticatedUser
   * @param changePassword
   */
  void changePassword(Long userId, String authenticatedUser, UserPassword changePassword);

  void authorizedGroups(String authenticatedUser, PersistableUser user);
  
  /**
   * Update user enable / disabled flag
   * @param store
   * @param user
   */
  void updateEnabled(MerchantStore store, PersistableUser user);
  
	/**
	 * 
	 * Forgot password functionality
	 * @param userName
	 * @param store
	 * @param language
	 */
	void requestPasswordReset(String userName, String userContextPath, MerchantStore store, Language language);
	
	/**
	 * Validates if a password request is valid
	 * @param token
	 * @param store
	 */
	void verifyPasswordRequestToken(String token, String store);
	
	
	/**
	 * Reset password
	 * @param password
	 * @param token
	 * @param store
	 */
	void resetPassword(String password, String token, String store);

}



```
