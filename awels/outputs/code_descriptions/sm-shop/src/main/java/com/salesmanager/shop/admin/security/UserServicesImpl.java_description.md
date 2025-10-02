# UserServicesImpl.java

## Review

## 1. Summary  

**Purpose**  
`UserServicesImpl` is a Spring‑managed service that supplies `UserDetails` to Spring Security and provides a helper for creating a default administrator account. It pulls user, group and permission data from the underlying core services and translates it into a `org.springframework.security.core.userdetails.User` instance that Spring Security can use for authentication and authorization.

**Key Components**

| Component | Responsibility |
|-----------|----------------|
| `UserService` | Retrieves `User` records from the database. |
| `MerchantStoreService` | Provides the default merchant store needed for a default admin. |
| `PermissionService` | Loads permissions for a set of groups. |
| `GroupService` | Retrieves group definitions (e.g. ADMIN). |
| `PasswordEncoder` | Hashes the default password when creating the super‑admin. |
| `WebUserServices` | (Not shown) Likely an interface that defines the `loadUserByUsername` method and `createDefaultAdmin`. |

**Design Patterns / Frameworks**

* **Spring MVC / Spring Data** – The class is annotated with `@Service` and uses `@Inject`/`@Named` for dependency injection.
* **Spring Security** – Implements `UserDetailsService` semantics by returning a `User` with authorities.
* **Factory / Adapter** – Adapts the core `User` model to Spring Security’s `UserDetails`.

---

## 2. Detailed Description  

### Execution Flow

1. **User lookup**  
   `loadUserByUsername(String userName)` is called by Spring Security during authentication.  
   * Calls `userService.getByUserName(userName)` to obtain the domain user.  
   * If the user is `null`, the method returns `null` (Spring Security interprets this as “user not found”).

2. **Authority assembly**  
   * A mandatory `ROLE_AUTHENTICATED` authority is added to every logged‑in user.  
   * The user's groups are iterated to collect their IDs.  
   * `permissionService.getPermissions(groupsId)` fetches all permissions belonging to those groups.  
   * Each permission name is prefixed with `ROLE_` and added to the authority list.

3. **UserDetails creation**  
   A `org.springframework.security.core.userdetails.User` is constructed with:
   * username
   * stored password (already hashed)
   * account status flags (active, account non‑expired, credentials non‑expired, account non‑locked)
   * the assembled authority list.

4. **Default admin creation**  
   `createDefaultAdmin()` is a manual bootstrap helper that:
   * Loads the default store.
   * Encodes the password “password”.
   * Finds all `ADMIN` groups, attaches those that are “SUPERADMIN” or “ADMIN”.
   * Persists a new user with email “admin@shopizer.com”.

### Assumptions & Constraints

* The domain `User` model stores an already‑hashed password (`getAdminPassword()`).
* Group names must match constants (`Constants.GROUP_SUPERADMIN` / `Constants.GROUP_ADMIN`) for the default admin.
* The method `getByUserName` may throw any exception; it is wrapped into a `SecurityDataAccessException`.
* It is assumed that Spring Security is configured to treat `ROLE_` prefixed authorities as roles.

### Architecture & Design Choices

* **Separation of Concerns** – Business logic for users, permissions, and groups is delegated to dedicated services; this class focuses only on bridging to Spring Security.
* **Explicit Authority Prefixing** – All permissions are converted into role names, simplifying the use of Spring Security’s `hasRole` checks.
* **Manual Default Admin Creation** – A one‑time helper method rather than an automated data‑seed script; it must be invoked by the developer or a startup script.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `loadUserByUsername(String userName)` | Implements Spring Security’s `UserDetailsService` lookup. | `userName` – login identifier. | `UserDetails` (or `null`). | Throws `UsernameNotFoundException` or `DataAccessException` on errors. |
| `createDefaultAdmin()` | Bootstrap helper to create the platform’s initial administrator. | None. | `void`. | Persists a new user; may throw `Exception`. |
| `getByUserName(String)` (from `UserService`) – called internally. | Retrieve a user by login. | `userName` | `User` or `null`. | No side effects. |
| `getPermissions(List<Integer> groupIds)` (from `PermissionService`) – called internally. | Load permissions for groups. | Group ID list. | `List<Permission>`. | No side effects. |
| `listGroup(GroupType)` (from `GroupService`) – called internally. | Load all groups of a type. | `GroupType` enum. | `List<Group>`. | No side effects. |

*Utility methods* – None defined explicitly; the class relies on injected services.

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `com.salesmanager.core.business.services.merchant.MerchantStoreService` | Third‑party service | Store retrieval |
| `com.salesmanager.core.business.services.user.*` | Third‑party services | User, group, permission CRUD |
| `org.springframework.security.*` | Spring Security | UserDetails, GrantedAuthority |
| `org.springframework.stereotype.Service` | Spring Framework | Bean registration |
| `javax.inject.Inject` / `javax.inject.Named` | Java EE CDI | Dependency injection |
| `org.slf4j.Logger` | SLF4J | Logging |
| `org.springframework.dao.DataAccessException` | Spring | Exception handling |
| `org.springframework.security.crypto.password.PasswordEncoder` | Spring Security | Password hashing |

All external dependencies are standard Spring/Spring‑Security libraries and the domain services provided by the “core” module of the Shopizer project.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation** – The class cleanly delegates to other services.  
* **Simplicity** – Minimal business logic, focused on bridging to Spring Security.  
* **Extensibility** – Adding new permissions only requires updating the permission table.

### Weaknesses & Edge Cases  

1. **Null handling** – Returning `null` for an unknown user is acceptable, but the method also catches *any* exception and returns a generic `SecurityDataAccessException`. This may hide the root cause (e.g., connection failures).  
2. **Password handling** – The method `createDefaultAdmin` hard‑codes the initial password (“password”). This is a security risk if the method is left in production code.  
3. **Role prefixing** – All permissions are treated as roles (`ROLE_` prefix). If a permission name already contains `ROLE_` this could double‑prefix and break checks.  
4. **Group ID extraction** – The loop that collects group IDs uses `group.getId()` but does not guard against `null`.  
5. **Transactional context** – `createDefaultAdmin` performs several operations but is not annotated with `@Transactional`; if any step fails, earlier changes may remain, leaving the system in an inconsistent state.  
6. **Error handling** – The helper method throws a generic `Exception`; callers may need finer‑grained control.

### Potential Enhancements  

| Area | Suggested Change |
|------|------------------|
| **Security** | Replace the hard‑coded password with a random, configurable initial password, or prompt the admin to change it on first login. |
| **Transactional** | Annotate `createDefaultAdmin` with `@Transactional` to ensure atomicity. |
| **Error handling** | Throw custom exceptions (`UserCreationException`) with descriptive messages. |
| **Logging** | Log successful admin creation, but avoid leaking sensitive data. |
| **Authority generation** | Allow a configuration flag to toggle whether to prefix permission names with `ROLE_`. |
| **Caching** | Cache group‑permission mappings to reduce DB load for frequent authentications. |
| **Tests** | Add unit tests for `loadUserByUsername` (mocking services) and integration tests for `createDefaultAdmin`. |
| **API exposure** | Expose a read‑only endpoint to list current user roles/permissions for audit purposes. |

Overall, the class fulfills its role within the Shopizer architecture but could benefit from tighter error handling, enhanced security practices, and transactional safety.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.admin.security;

import com.salesmanager.core.business.services.merchant.MerchantStoreService;
import com.salesmanager.core.business.services.user.GroupService;
import com.salesmanager.core.business.services.user.PermissionService;
import com.salesmanager.core.business.services.user.UserService;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.user.Group;
import com.salesmanager.core.model.user.GroupType;
import com.salesmanager.core.model.user.Permission;
import com.salesmanager.shop.constants.Constants;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataAccessException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

import javax.inject.Inject;
import javax.inject.Named;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;


/**
 * 
 * @author casams1
 *         http://stackoverflow.com/questions/5105776/spring-security-with
 *         -custom-user-details
 */
@Service("userDetailsService")
public class UserServicesImpl implements WebUserServices{
	
	private static final Logger LOGGER = LoggerFactory.getLogger(UserServicesImpl.class);
	
	private static final String DEFAULT_INITIAL_PASSWORD = "password";

	@Inject
	private UserService userService;
	

	@Inject
	private MerchantStoreService merchantStoreService;
	
	@Inject
	@Named("passwordEncoder")
	private PasswordEncoder passwordEncoder;
	

	
	@Inject
	protected PermissionService  permissionService;
	
	@Inject
	protected GroupService   groupService;
	
	public final static String ROLE_PREFIX = "ROLE_";
	
	
	
	public UserDetails loadUserByUsername(String userName)
			throws UsernameNotFoundException, DataAccessException {

		com.salesmanager.core.model.user.User user = null;
		Collection<GrantedAuthority> authorities = new ArrayList<GrantedAuthority>();
		
		try {

			user = userService.getByUserName(userName);

			if(user==null) {
				return null;
			}

			GrantedAuthority role = new SimpleGrantedAuthority(ROLE_PREFIX + Constants.PERMISSION_AUTHENTICATED);//required to login
			authorities.add(role);
	
			List<Integer> groupsId = new ArrayList<Integer>();
			List<Group> groups = user.getGroups();
			for(Group group : groups) {
				
				
				groupsId.add(group.getId());
				
			}
			
	
	    	
	    	List<Permission> permissions = permissionService.getPermissions(groupsId);
	    	for(Permission permission : permissions) {
	    		GrantedAuthority auth = new SimpleGrantedAuthority(ROLE_PREFIX + permission.getPermissionName());
	    		authorities.add(auth);
	    	}
    	
		} catch (Exception e) {
			LOGGER.error("Exception while querrying user",e);
			throw new SecurityDataAccessException("Exception while querrying user",e);
		}
		
		
		
	
		
		User secUser = new User(userName, user.getAdminPassword(), user.isActive(), true,
				true, true, authorities);
		return secUser;
	}
	
	
	public void createDefaultAdmin() throws Exception {

		  MerchantStore store = merchantStoreService.getByCode(MerchantStore.DEFAULT_STORE);

		  String password = passwordEncoder.encode(DEFAULT_INITIAL_PASSWORD);
		  
		  List<Group> groups = groupService.listGroup(GroupType.ADMIN);
		  
		  //creation of the super admin admin:password)
		  com.salesmanager.core.model.user.User user = new com.salesmanager.core.model.user.User("admin@shopizer.com",password,"admin@shopizer.com");
		  user.setFirstName("Administrator");
		  user.setLastName("User");
		  
		  for(Group group : groups) {
			  if(group.getGroupName().equals(Constants.GROUP_SUPERADMIN) || group.getGroupName().equals(Constants.GROUP_ADMIN)) {
				  user.getGroups().add(group);
			  }
		  }

		  user.setMerchantStore(store);		  
		  userService.create(user);
		
		
	}



}



```
