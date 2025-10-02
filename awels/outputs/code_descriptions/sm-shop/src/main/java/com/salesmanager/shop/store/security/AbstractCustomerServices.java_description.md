# AbstractCustomerServices.java

## Review

## 1. Summary  

`AbstractCustomerServices` is a reusable, Spring‑Security‑centric implementation that loads a **Customer** by nickname, constructs a `UserDetails` instance and wires together the necessary authorities (roles and permissions).  

* **Key responsibilities**  
  * Fetch the `Customer` record from the database via `CustomerService`.  
  * Assemble a list of `GrantedAuthority` objects: a mandatory *authenticated* role plus any permissions that belong to the customer’s groups.  
  * Delegate the creation of the concrete `UserDetails` object to a concrete subclass through the abstract `userDetails(...)` method.  

* **Design patterns & frameworks**  
  * **Template Method** – the abstract method `userDetails` lets subclasses decide the exact shape of `UserDetails` while reusing the lookup logic.  
  * **Spring Security** – implements `UserDetailsService` to plug into the authentication workflow.  
  * **Dependency Injection** – constructor‑based injection of the three domain services.  

---

## 2. Detailed Description  

### Core Flow  

1. **Invocation** – Spring Security calls `loadUserByUsername(userName)`.  
2. **Lookup** – The method logs the request and calls `customerService.getByNick(userName)`.  
3. **Error handling** –  
   * If no customer is returned → `UsernameNotFoundException` is thrown.  
   * Any `ServiceException` from the domain services is wrapped in a `SecurityDataAccessException`.  
4. **Authority construction** –  
   * A mandatory role `"ROLE_CUSTOMER_AUTHENTICATED"` is added.  
   * The customer’s group IDs are extracted and used to fetch all associated permissions via `permissionService.getPermissions(groupsId)`.  
   * Each permission name becomes a `SimpleGrantedAuthority` and is appended to the collection.  
5. **UserDetails creation** – The subclass‑implemented `userDetails` method receives the username, the `Customer` entity, and the authorities, and returns the concrete `UserDetails` instance that Spring Security will use for authentication.  

### Assumptions & Constraints  

* A customer is uniquely identified by its *nickname*.  
* `Customer.getGroups()` returns a non‑null collection (or the code must guard against `null`).  
* Permission names are already prefixed appropriately (`ROLE_…`) or not; the code simply concatenates `ROLE_PREFIX` to the *customer authenticated* role only.  
* The returned authorities may contain duplicates; no deduplication is performed.  
* The code runs within a Spring context with proper bean wiring for the three services.  

### Architectural Choices  

* **Template Method** simplifies common logic while allowing flexibility in the `UserDetails` representation.  
* **Using `CollectionUtils.isNotEmpty`** from Apache Commons offers a null‑safe check.  
* **`List<GrantedAuthority>`** rather than a `Set` keeps insertion order, though it may lead to duplicates.  

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `public AbstractCustomerServices(CustomerService, PermissionService, GroupService)` | Constructor – injects domain services. | `customerService`, `permissionService`, `groupService` | N/A | Stores services in protected fields. |
| `protected abstract UserDetails userDetails(String userName, Customer customer, Collection<GrantedAuthority> authorities)` | Subclass hook – creates a concrete `UserDetails`. | `userName`, `customer`, `authorities` | `UserDetails` | None – must be implemented by subclass. |
| `public UserDetails loadUserByUsername(String userName)` | Spring Security contract – loads user data, assembles authorities, delegates to `userDetails`. | `userName` | `UserDetails` | May throw `UsernameNotFoundException`, `DataAccessException`; logs debug/info; may throw `SecurityDataAccessException`. |

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.security.*` | Third‑party (Spring Security) | Provides `UserDetailsService`, `GrantedAuthority`, etc. |
| `org.apache.commons.collections4.CollectionUtils` | Third‑party (Apache Commons Collections) | Null‑safe collection utilities. |
| `org.slf4j.*` | Logging façade | For debug and error logging. |
| `com.salesmanager.core.business.services.*` | Internal | Domain services for customers, permissions, groups. |
| `com.salesmanager.core.model.*` | Internal | Domain entities (`Customer`, `Group`, `Permission`). |
| `com.salesmanager.shop.admin.security.SecurityDataAccessException` | Internal | Wraps data‑access errors for Spring Security. |
| `com.salesmanager.shop.constants.Constants` | Internal | Holds constants like `PERMISSION_CUSTOMER_AUTHENTICATED`. |
| `org.springframework.dao.DataAccessException` | Spring | Generic data‑access exception used by the interface. |

All dependencies are expected to be resolved in a Maven/Gradle build and are typical for a Spring MVC application.

---

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Issues  

1. **`customer.getGroups()` could be `null`** – currently leads to `NullPointerException`.  
   * *Fix*: Guard with `Optional.ofNullable(user.getGroups()).orElse(Collections.emptyList())`.  
2. **Duplicate authorities** – If a customer belongs to multiple groups that share the same permission, the resulting list contains duplicates.  
   * *Fix*: Switch to a `Set<GrantedAuthority>` or de‑duplicate after construction.  
3. **Missing permission names** – If a `Permission` entity has a `null` or empty `permissionName`, the resulting `SimpleGrantedAuthority` may be invalid.  
   * *Fix*: Validate and skip such entries.  
4. **Hardcoded role prefix** – `ROLE_PREFIX` is hardcoded; if the application uses a different prefix elsewhere, inconsistencies may arise.  
   * *Fix*: Externalize to a configuration property.  

### Future Enhancements  

* **Caching** – Frequently accessed customer details and permissions could be cached (e.g., with Spring Cache) to reduce database load.  
* **Transactional boundaries** – If the lookup requires multiple DB calls, consider wrapping in a read‑only transaction to avoid partial reads.  
* **Unit tests** – Mock the three services and verify authority assembly, error handling, and subclass delegation.  
* **Extensibility** – Provide a default implementation of `userDetails` that constructs a standard `org.springframework.security.core.userdetails.User`, allowing simple concrete classes to just call `super`.  
* **Security hardening** – Verify that permission names cannot contain malicious characters that could interfere with role resolution or URL matching.

Overall, the class offers a clean, reusable foundation for customer authentication within the application’s Spring Security pipeline. With a few defensive‑coding improvements and optional caching, it can serve as a robust backbone for future authentication extensions.

## Code Critique



## Code Preview

```java
package com.salesmanager.shop.store.security;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

import org.apache.commons.collections4.CollectionUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataAccessException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.customer.CustomerService;
import com.salesmanager.core.business.services.user.GroupService;
import com.salesmanager.core.business.services.user.PermissionService;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.user.Group;
import com.salesmanager.core.model.user.Permission;
import com.salesmanager.shop.admin.security.SecurityDataAccessException;
import com.salesmanager.shop.constants.Constants;

public abstract class AbstractCustomerServices implements UserDetailsService{
	
	private static final Logger LOGGER = LoggerFactory.getLogger(AbstractCustomerServices.class);
	
	protected CustomerService customerService;
	protected PermissionService  permissionService;
	protected GroupService   groupService;
	
	public final static String ROLE_PREFIX = "ROLE_";//Spring Security 4
	
	public AbstractCustomerServices(
			CustomerService customerService, 
			PermissionService permissionService, 
			GroupService groupService) {
		
		this.customerService = customerService;
		this.permissionService = permissionService;
		this.groupService = groupService;
	}
	
	protected abstract UserDetails userDetails(String userName, Customer customer, Collection<GrantedAuthority> authorities);
	

	public UserDetails loadUserByUsername(String userName)
			throws UsernameNotFoundException, DataAccessException {
		Customer user = null;
		Collection<GrantedAuthority> authorities = new ArrayList<GrantedAuthority>();

		try {
			
				LOGGER.debug("Loading user by user id: {}", userName);

				user = customerService.getByNick(userName);
			
				if(user==null) {
					//return null;
					throw new UsernameNotFoundException("User " + userName + " not found");
				}
	
	

			GrantedAuthority role = new SimpleGrantedAuthority(ROLE_PREFIX + Constants.PERMISSION_CUSTOMER_AUTHENTICATED);//required to login
			authorities.add(role); 
			
			List<Integer> groupsId = new ArrayList<Integer>();
			List<Group> groups = user.getGroups();
			for(Group group : groups) {
				groupsId.add(group.getId());
			}
			
	
			if(CollectionUtils.isNotEmpty(groupsId)) {
		    	List<Permission> permissions = permissionService.getPermissions(groupsId);
		    	for(Permission permission : permissions) {
		    		GrantedAuthority auth = new SimpleGrantedAuthority(permission.getPermissionName());
		    		authorities.add(auth);
		    	}
			}
			

		} catch (ServiceException e) {
			LOGGER.error("Exception while querrying customer",e);
			throw new SecurityDataAccessException("Cannot authenticate customer",e);
		}

		return userDetails(userName, user, authorities);
		
	}

}



```
