# CustomerServiceImpl.java

## Review

## 1. Summary
`CustomerServiceImpl` is a Spring‐managed service that encapsulates all customer‑related business logic for the SalesManager application.  
* **Core responsibilities**  
  * Query customers by name, id, nickname, or reset‑token.  
  * Persist (create/update) customer entities.  
  * Delete customers and cascade‑delete their custom attributes.  
  * Resolve a customer’s address from an IP address via a `GeoLocation` helper.  
* **Key components**  
  * `CustomerRepository` – Spring Data repository that performs the actual data access.  
  * `CustomerAttributeService` – Service used to manage customer attributes (deleted when a customer is removed).  
  * `GeoLocation` – Utility that converts an IP address into an `Address`.  
  * `SalesManagerEntityServiceImpl` – Generic CRUD base class that provides `create`, `update`, `delete`, etc.  
* **Design patterns / frameworks**  
  * **Service Layer** – The class acts as a façade over the persistence layer.  
  * **Dependency Injection** – Spring’s `@Service`, `@Inject` annotations wire the dependencies.  
  * **DAO / Repository** – `CustomerRepository` follows Spring Data JPA conventions.  
  * **Exception Handling** – Wraps lower‑level exceptions in a domain‑specific `ServiceException`.  

## 2. Detailed Description
1. **Initialization**  
   * The constructor receives a `CustomerRepository` and passes it to the superclass.  
   * `CustomerAttributeService` and `GeoLocation` are injected by Spring.  
2. **Runtime behaviour**  
   * **Read operations** delegate to the repository (e.g. `findByName`, `findByNick`, `listByStore`).  
   * **Create/Update**:  
     * If an ID is present, `super.update()` is called; otherwise `super.create()` is invoked.  
   * **Delete**:  
     * Re‑retrieves the customer (ensuring it exists).  
     * Loads all `CustomerAttribute` instances for that customer and deletes each via `customerAttributeService`.  
     * Finally calls `customerRepository.delete(customer)`.  
   * **Address resolution**: calls `geoLocation.getAddress(ip)` and wraps any exception in `ServiceException`.  
3. **Cleanup** – None needed; Spring handles bean lifecycle.  
4. **Assumptions / constraints**  
   * A customer must always belong to a `MerchantStore`; many methods rely on this (e.g., attribute deletion).  
   * Repository methods are expected to throw unchecked exceptions; the service layer only wraps the address resolution exception.  
   * The service is stateless and thread‑safe (Spring beans are singletons by default).  

## 3. Functions/Methods
| Method | Purpose | Inputs | Output | Side‑Effects |
|--------|---------|--------|--------|--------------|
| `getByName(String firstName)` | Fetch all customers with a given first name. | `firstName` | `List<Customer>` | None |
| `getById(Long id)` | Retrieve a customer by primary key. | `id` | `Customer` | None |
| `getByNick(String nick)` | Retrieve a customer by nickname (global). | `nick` | `Customer` | None |
| `getByNick(String nick, int storeId)` | Retrieve a customer by nickname within a specific store. | `nick`, `storeId` | `Customer` | None |
| `getListByStore(MerchantStore store)` | List all customers for a store. | `store` | `List<Customer>` | None |
| `getListByStore(MerchantStore store, CustomerCriteria criteria)` | Paginated/filtered list of customers. | `store`, `criteria` | `CustomerList` | None |
| `getCustomerAddress(MerchantStore store, String ipAddress)` | Resolve an IP to an `Address` via GeoLocation. | `store`, `ipAddress` | `Address` | May throw `ServiceException` |
| `saveOrUpdate(Customer customer)` | Persist a new or existing customer. | `customer` | None | Calls `super.create()` or `super.update()` |
| `delete(Customer customer)` | Remove a customer and all its attributes. | `customer` | None | Deletes attributes, then customer |
| `getByNick(String nick, String code)` | Retrieve by nickname scoped to store code. | `nick`, `code` | `Customer` | None |
| `getByPasswordResetToken(String storeCode, String token)` | Find customer by password‑reset token. | `storeCode`, `token` | `Customer` | None |

**Reusable / Utility Methods**  
The class itself mainly delegates; the only small utility is the check `customer.getId()!=null && customer.getId()>0` to decide between create/update.

## 4. Dependencies
| Dependency | Type | Role |
|------------|------|------|
| `org.springframework.stereotype.Service` | Framework | Marks bean for Spring component scanning. |
| `javax.inject.Inject` | Framework | Injects collaborators. |
| `org.slf4j.Logger` / `LoggerFactory` | Logging | Records debug messages. |
| `com.salesmanager.core.business.repositories.customer.CustomerRepository` | Custom | Spring Data JPA repository. |
| `com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl` | Custom | Generic CRUD base class. |
| `com.salesmanager.core.business.services.customer.attribute.CustomerAttributeService` | Custom | Manages customer attributes. |
| `com.salesmanager.core.modules.utils.GeoLocation` | Custom | IP→Address resolution. |
| `com.salesmanager.core.model.*` | Domain models | `Customer`, `Address`, `MerchantStore`, etc. |
| `com.salesmanager.core.business.exception.ServiceException` | Custom | Domain‑specific exception. |

All dependencies are either standard Java/SLF4J/Spring or project‑specific; there are no external third‑party libraries beyond Spring Data JPA and SLF4J.

## 5. Additional Notes
### Strengths
* **Clear separation of concerns** – Service delegates persistence to repository, keeps business logic minimal.  
* **Use of generic base class** – Avoids boilerplate CRUD code.  
* **Dependency injection** – Enhances testability.  
* **Exception handling** – Wraps IP resolution errors, exposing a consistent exception type to callers.

### Potential Issues / Edge Cases
1. **Null Checks** – Methods like `getByName`, `getByNick` assume the repository returns non‑null lists; callers should be prepared for empty results.  
2. **Race Condition on Delete** – Between `getById` and `customerRepository.delete`, another thread could modify the customer. Transactional boundaries (not shown) would mitigate this.  
3. **Missing @Transactional** – The class does not declare transactional semantics. Without it, multiple database calls (e.g., attribute deletion + customer deletion) might not roll back together on failure.  
4. **Hard‑coded ID check** – `customer.getId()!=null && customer.getId()>0` assumes negative IDs are invalid; a clearer `customer.isPersisted()` helper would be cleaner.  
5. **Exception Transparency** – Only the IP resolution method wraps exceptions; other methods expose repository exceptions directly. Consistency would improve error handling.  
6. **Method Overloading Ambiguity** – Three `getByNick` overloads could confuse callers; Javadoc or more descriptive method names could help.  

### Future Enhancements
* **Transactional Annotation** – Annotate service methods with `@Transactional` to ensure atomic operations (especially delete).  
* **Input Validation** – Add validation (e.g., non‑blank nickname) and throw `IllegalArgumentException` or a custom validation exception.  
* **Caching** – Frequently accessed customer data (by ID or nickname) could be cached to reduce DB load.  
* **Bulk Attribute Delete** – Use a single repository call to delete attributes instead of looping.  
* **Unified Error Handling** – Create a private helper to wrap repository exceptions in `ServiceException`.  
* **Unit Tests** – Mock `CustomerRepository`, `CustomerAttributeService`, and `GeoLocation` to test each method in isolation.  

Overall, the implementation follows common Spring patterns and is straightforward. Addressing the above edge cases would make it more robust and production‑ready.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.customer;

import java.util.List;

import javax.inject.Inject;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.customer.CustomerRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.business.services.customer.attribute.CustomerAttributeService;
import com.salesmanager.core.model.common.Address;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.customer.CustomerCriteria;
import com.salesmanager.core.model.customer.CustomerList;
import com.salesmanager.core.model.customer.attribute.CustomerAttribute;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.modules.utils.GeoLocation;



@Service("customerService")
public class CustomerServiceImpl extends SalesManagerEntityServiceImpl<Long, Customer> implements CustomerService {

	private static final Logger LOGGER = LoggerFactory.getLogger(CustomerServiceImpl.class);
	
	private CustomerRepository customerRepository;
	
	@Inject
	private CustomerAttributeService customerAttributeService;
	
	@Inject
	private GeoLocation geoLocation;

	
	@Inject
	public CustomerServiceImpl(CustomerRepository customerRepository) {
		super(customerRepository);
		this.customerRepository = customerRepository;
	}

	@Override
	public List<Customer> getByName(String firstName) {
		return customerRepository.findByName(firstName);
	}
	
	@Override
	public Customer getById(Long id) {
			return customerRepository.findOne(id);		
	}
	
	@Override
	public Customer getByNick(String nick) {
		return customerRepository.findByNick(nick);	
	}
	
	@Override
	public Customer getByNick(String nick, int storeId) {
		return customerRepository.findByNick(nick, storeId);	
	}
	
	@Override
	public List<Customer> getListByStore(MerchantStore store) {
		return customerRepository.findByStore(store.getId());
	}
	
	@Override
	public CustomerList getListByStore(MerchantStore store, CustomerCriteria criteria) {
		return customerRepository.listByStore(store,criteria);
	}
	
	@Override
	public Address getCustomerAddress(MerchantStore store, String ipAddress) throws ServiceException {
		
		try {
			return geoLocation.getAddress(ipAddress);
		} catch(Exception e) {
			throw new ServiceException(e);
		}
		
	}

	@Override	
	public void saveOrUpdate(Customer customer) throws ServiceException {

		LOGGER.debug("Creating Customer");
		
		if(customer.getId()!=null && customer.getId()>0) {
			super.update(customer);
		} else {			
		
			super.create(customer);

		}
	}

	public void delete(Customer customer) throws ServiceException {
		customer = getById(customer.getId());
		
		//delete attributes
		List<CustomerAttribute> attributes =customerAttributeService.getByCustomer(customer.getMerchantStore(), customer);
		if(attributes!=null) {
			for(CustomerAttribute attribute : attributes) {
				customerAttributeService.delete(attribute);
			}
		}
		customerRepository.delete(customer);

	}

	@Override
	public Customer getByNick(String nick, String code) {
		return customerRepository.findByNick(nick, code);
	}

	@Override
	public Customer getByPasswordResetToken(String storeCode, String token) {
		return customerRepository.findByResetPasswordToken(token, storeCode);
	}
	

}



```
