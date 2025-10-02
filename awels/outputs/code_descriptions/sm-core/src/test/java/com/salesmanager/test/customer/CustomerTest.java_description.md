# CustomerTest.java

## Review

## 1. Summary  

**Purpose**  
The file defines a JUnit integration test (`CustomerTest`) that exercises the customer‑creation workflow in the **SalesManager** core engine. The test builds a `Customer` entity together with its billing/delivery addresses, a set of `CustomerOption`/`CustomerOptionValue` objects (e.g., “subscribed to mailing list”), and finally attaches a `CustomerAttribute` to the customer before persisting and cleaning up.

**Key components**  

| Component | Role |
|-----------|------|
| `CustomerService` | CRUD for `Customer` |
| `CustomerOptionService` | CRUD for `CustomerOption` |
| `CustomerOptionValueService` | CRUD for `CustomerOptionValue` |
| `CustomerOptionSetService` | CRUD for the mapping of an option to its possible values |
| `MerchantStore`, `Country`, `Zone`, `Language` | Reference data injected into the domain objects |
| `AbstractSalesManagerCoreTestCase` | Base class that wires the service beans (Spring) and provides transactional support |

**Design patterns / frameworks**  

* **Spring** – dependency injection for services; transaction management via the base test case.  
* **DAO / Service layer** – business logic is encapsulated in services; the test interacts only with the service APIs.  
* **Domain‑Driven Design** – rich domain objects (`Customer`, `Billing`, `Delivery`, `CustomerOption`, …) with encapsulated state.  
* **JUnit 4** – test framework; `@Ignore` disables the test by default.  

---

## 2. Detailed Description  

### Execution flow  

1. **Setup**  
   * `@Ignore` prevents the test from running automatically.  
   * The base test case (`AbstractSalesManagerCoreTestCase`) supplies autowired services and starts a transaction.

2. **Domain construction**  
   * Retrieve reference entities (`Language`, `MerchantStore`, `Country`, `Zone`).  
   * Create a new `Customer` with basic attributes, billing and delivery addresses.  
   * Persist the customer via `customerService.create(customer)`.

3. **Option/value creation**  
   * Build two `CustomerOptionValue` instances (`yes`, `no`) with descriptions in English.  
   * Persist each value via `customerOptionValueService.create(...)`.

4. **Option definition**  
   * Create two `CustomerOption` objects (`subscribedToMailingList`, `hasReturnedItems`) with descriptions.  
   * Persist them using `customerOptionService.create(...)`.  
   * Update the sort order after creation.

5. **Option set (value association)**  
   * For each option, create `CustomerOptionSet` instances that bind the option to a particular value (`yes`/`no`).  
   * Persist the sets via `customerOptionSetService.create(...)`.

6. **Attribute linking**  
   * Instantiate a `CustomerAttribute` that associates the customer with a specific option/value pair (`subscribedToMailingList` + `no`).  
   * Attach it to the customer’s `attributes` collection.  
   * Persist the updated customer (`customerService.save(customer)`).

7. **Cleanup**  
   * Delete the customer via `customerService.delete(customer)`.  
   * The surrounding transaction (managed by the base test case) rolls back unless the delete is explicitly committed.

### Assumptions & constraints  

| Assumption | Explanation |
|------------|-------------|
| All reference data (`en`, `CA`, `QC`) exist in the test database. | The test will fail if any of these codes are missing. |
| Service layer uses the default Spring transaction (REQUIRED). | The test runs inside a single transaction that is rolled back after the test, ensuring no persistence leak. |
| `CustomerOption` and `CustomerOptionValue` are unique per store. | The test does not check for duplicates; it will fail if the same codes already exist. |
| No external state changes are needed after the test. | Customer is deleted at the end, leaving the database clean. |

### Architecture & design choices  

* **Integration‑style test** – Rather than mocking services, the test exercises the full persistence stack (Spring + JPA/Hibernate).  
* **Explicit ordering of entity creation** – Option values are persisted before options that reference them, and option sets are created after both the option and its value exist.  
* **Use of `@Ignore`** – Indicates that the test is currently disabled, possibly because it is slow or requires a specific data set.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `createCustomer()` | Main test routine that creates a customer with options and attributes, then cleans up. | None (uses hard‑coded data). | None. | Persists a `Customer` and related entities; deletes the customer at the end. |

### Supporting helper logic (inside `createCustomer`)  

| Helper block | What it does |
|--------------|--------------|
| **Retrieve reference entities** | `languageService.getByCode("en")`, `merchantService.getByCode(...)`, etc. |
| **Build `Customer`** | Sets email, gender, anonymous flag, etc.; attaches `Billing` & `Delivery`. |
| **Persist option values** | `customerOptionValueService.create(...)` for `yes` and `no`. |
| **Build & persist options** | `CustomerOption` instances, add descriptions, then `create`. |
| **Create option sets** | Bind an option to a value; persist via `customerOptionSetService.create(...)`. |
| **Link attribute to customer** | `CustomerAttribute` creation and addition to `customer.getAttributes()`. |
| **Save & delete** | `customerService.save(customer)` then `customerService.delete(customer)`. |

---

## 4. Dependencies  

| External library / framework | Nature | Notes |
|------------------------------|--------|-------|
| **Spring Framework** | Third‑party | Provides dependency injection, transaction management, and data access (likely JPA/Hibernate). |
| **JUnit 4** | Third‑party | Test framework; `@Test`, `@Ignore`. |
| **SalesManager Core modules** | Third‑party | Domain classes (`Customer`, `CustomerOption`, etc.) and services (`CustomerService`, etc.). |
| **Hibernate / JPA** | Third‑party | Persist entities; not explicitly referenced in the test code but implied by service layer. |
| **Java Standard Library** | Standard | `java.util.*`, `java.lang.*`. |

No platform‑specific dependencies are evident; the code should run on any JVM that supports the used libraries.

---

## 5. Additional Notes  

### Strengths  
* **Full end‑to‑end coverage** – The test validates that all related services cooperate correctly to persist a complex customer object.  
* **Clear, step‑by‑step construction** – Each logical block (values, options, sets, attributes) is isolated and commented, making the test understandable.  

### Potential issues / edge cases  

1. **Hard‑coded data** – The test will fail if any of the reference codes (`en`, `CA`, `QC`, store code) are missing.  
2. **Lack of assertions** – The test prints the size of the option set list but does not verify expected outcomes. A true unit test should assert that the number of option sets equals 3 and that the saved customer contains the expected attribute.  
3. **Race conditions in a shared database** – Running this test concurrently with others that create the same option/value codes could result in constraint violations.  
4. **Transactional side‑effects** – Although the base test case likely rolls back, the explicit delete (`customerService.delete(customer)`) may interfere with that rollback if not handled properly.  
5. **Ignored test** – With `@Ignore`, developers may overlook that this test is out of date or requires manual activation.  

### Suggested improvements  

| Improvement | Why | How |
|-------------|-----|-----|
| **Remove `@Ignore` and add real assertions** | Ensures the test actually verifies behavior. | Use `assertEquals`, `assertNotNull`, etc. to check that the customer, options, and attributes are persisted correctly. |
| **Parameterize reference data** | Avoids hard‑coding and allows running against multiple locales or stores. | Use a data provider or Spring’s `@TestPropertySource`. |
| **Use `@Before` / `@After` for setup/cleanup** | Cleaner separation of concerns; easier to maintain. | Move entity creation into a helper method invoked by `@Before`, and deletion into `@After`. |
| **Mock services for faster unit tests** | If only the service layer logic is needed, mocking reduces dependency on the database. | Use Mockito to stub repository calls. |
| **Add uniqueness checks** | Prevent duplicate creation during test runs. | Call `customerOptionService.findByCodeAndStore(...)` before creation. |
| **Document business rules** | Clarify why certain sort orders are set. | Add comments or a separate README snippet. |

---

**Conclusion**  
The test provides a solid integration scenario for creating customers with options and attributes in the SalesManager system. Enhancing it with assertions, parameterization, and a clearer lifecycle will turn it into a reliable regression test that can be run continuously without manual intervention.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.customer;

import java.util.Date;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import org.junit.Ignore;
import org.junit.Test;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.model.common.Billing;
import com.salesmanager.core.model.common.Delivery;
import com.salesmanager.core.model.customer.Customer;
import com.salesmanager.core.model.customer.CustomerGender;
import com.salesmanager.core.model.customer.attribute.CustomerAttribute;
import com.salesmanager.core.model.customer.attribute.CustomerOption;
import com.salesmanager.core.model.customer.attribute.CustomerOptionDescription;
import com.salesmanager.core.model.customer.attribute.CustomerOptionSet;
import com.salesmanager.core.model.customer.attribute.CustomerOptionValue;
import com.salesmanager.core.model.customer.attribute.CustomerOptionValueDescription;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.country.Country;
import com.salesmanager.core.model.reference.language.Language;
import com.salesmanager.core.model.reference.zone.Zone;



@Ignore
public class CustomerTest extends com.salesmanager.test.common.AbstractSalesManagerCoreTestCase {
	
	@Test
	public void createCustomer() throws ServiceException {
		
		
		Language en = languageService.getByCode("en");
		
		
		MerchantStore store = merchantService.getByCode(MerchantStore.DEFAULT_STORE);
		Country country = countryService.getByCode("CA");
		Zone zone = zoneService.getByCode("QC");
		
		/** Core customer attributes **/
		Customer customer = new Customer();
		customer.setMerchantStore(store);
		customer.setEmailAddress("test@test.com");
		customer.setGender(CustomerGender.M);

		customer.setAnonymous(true);
		customer.setCompany("ifactory");
		customer.setDateOfBirth(new Date());
		customer.setNick("My nick");
		customer.setPassword("123456");
		customer.setDefaultLanguage(store.getDefaultLanguage());
		
	    Delivery delivery = new Delivery();
	    delivery.setAddress("Shipping address");
	    delivery.setCountry(country);
	    delivery.setZone(zone);
	    
	    
	    Billing billing = new Billing();
	    billing.setFirstName("John");
	    billing.setLastName("Bossanova");
	    billing.setAddress("Billing address");
	    billing.setCountry(country);
	    billing.setZone(zone);
	    
	    customer.setBilling(billing);
	    customer.setDelivery(delivery);
		
		customerService.create(customer);
		customer = customerService.getById(customer.getId());
		

		//create an option value
		CustomerOptionValue yes = new CustomerOptionValue();
		yes.setCode("yes");
		yes.setMerchantStore(store);
		CustomerOptionValueDescription yesDescription = new CustomerOptionValueDescription();
		yesDescription.setLanguage(en);
		yesDescription.setCustomerOptionValue(yes);
		
		CustomerOptionValueDescription yes_sir = new CustomerOptionValueDescription();
		yes_sir.setCustomerOptionValue(yes);
		yes_sir.setDescription("Yes sir!");
		yes_sir.setName("Yes sir!");
		yes_sir.setLanguage(en);
		yes.getDescriptions().add(yes_sir);
		
		//needs to be saved before using it
		customerOptionValueService.create(yes);
		
		CustomerOptionValue no = new CustomerOptionValue();
		no.setCode("no");
		no.setMerchantStore(store);
		CustomerOptionValueDescription noDescription = new CustomerOptionValueDescription();
		noDescription.setLanguage(en);
		noDescription.setCustomerOptionValue(no);
		
		CustomerOptionValueDescription no_sir = new CustomerOptionValueDescription();
		no_sir.setCustomerOptionValue(no);
		no_sir.setDescription("Nope!");
		no_sir.setName("Nope!");
		no_sir.setLanguage(en);
		no.getDescriptions().add(no_sir);
		
		//needs to be saved before using it
		customerOptionValueService.create(no);
		
		
		//create a customer option to be used
		CustomerOption subscribedToMailingList = new CustomerOption();
		subscribedToMailingList.setActive(true);
		subscribedToMailingList.setPublicOption(true);
		subscribedToMailingList.setCode("subscribedToMailingList");
		subscribedToMailingList.setMerchantStore(store);
		
		CustomerOptionDescription mailingListDesciption= new CustomerOptionDescription();
		mailingListDesciption.setName("Subscribed to mailing list");
		mailingListDesciption.setDescription("Subscribed to mailing list");
		mailingListDesciption.setLanguage(en);
		mailingListDesciption.setCustomerOption(subscribedToMailingList);
		
		Set<CustomerOptionDescription> mailingListDesciptionList = new HashSet<CustomerOptionDescription>();
		mailingListDesciptionList.add(mailingListDesciption);
		subscribedToMailingList.setDescriptions(mailingListDesciptionList);
		
		customerOptionService.create(subscribedToMailingList);
		
		
		//create a customer option to be used
		CustomerOption hasReturnedItems = new CustomerOption();
		hasReturnedItems.setActive(true);
		hasReturnedItems.setPublicOption(true);
		hasReturnedItems.setCode("hasReturnedItems");
		hasReturnedItems.setMerchantStore(store);
		
		CustomerOptionDescription hasReturnedItemsDesciption= new CustomerOptionDescription();
		hasReturnedItemsDesciption.setName("Has returned items");
		hasReturnedItemsDesciption.setDescription("Has returned items");
		hasReturnedItemsDesciption.setLanguage(en);
		hasReturnedItemsDesciption.setCustomerOption(hasReturnedItems);
		
		Set<CustomerOptionDescription> hasReturnedItemsList = new HashSet<CustomerOptionDescription>();
		hasReturnedItemsList.add(hasReturnedItemsDesciption);
		hasReturnedItems.setDescriptions(hasReturnedItemsList);

		customerOptionService.create(hasReturnedItems);
		
		subscribedToMailingList.setSortOrder(3);
		
		customerOptionService.update(subscribedToMailingList);
		
		//--
		//now create an option set (association of a customer option with possible customer option values)
		//--
		
		//possible yes
		CustomerOptionSet mailingListSetYes = new CustomerOptionSet();

		mailingListSetYes.setSortOrder(0);
		mailingListSetYes.setCustomerOption(subscribedToMailingList);
		mailingListSetYes.setCustomerOptionValue(yes);
		
		customerOptionSetService.create(mailingListSetYes);

		//possible no
		CustomerOptionSet mailingListSetNo = new CustomerOptionSet();
		//mailingListSetNo.setPk(mailingListSetNoId);
		mailingListSetNo.setSortOrder(1);
		mailingListSetNo.setCustomerOption(subscribedToMailingList);
		mailingListSetNo.setCustomerOptionValue(no);
		
		customerOptionSetService.create(mailingListSetNo);
		
		//possible has returned items
		
		CustomerOptionSet hasReturnedItemsYes = new CustomerOptionSet();
		hasReturnedItemsYes.setSortOrder(0);
		hasReturnedItemsYes.setCustomerOption(hasReturnedItems);
		hasReturnedItemsYes.setCustomerOptionValue(yes);
		
		customerOptionSetService.create(hasReturnedItemsYes);
		
		
		subscribedToMailingList.setSortOrder(2);
		customerOptionService.update(subscribedToMailingList);
		
		CustomerOption option = customerOptionService.getById(subscribedToMailingList.getId());
		
		option.setSortOrder(4);
		customerOptionService.update(option);
		
		List<CustomerOptionSet> optionSetList = customerOptionSetService.listByStore(store, en);
		
		//Assert.assertEquals(3, optionSetList.size());
		System.out.println("Size of options : " + optionSetList.size());
		
		/**
		 * Now create a customer option attribute
		 * A customer attribute is a selected customer option set transformed to an 
		 * attribute for a given customer
		 */
		
		CustomerAttribute customerAttributeMailingList = new CustomerAttribute();
		customerAttributeMailingList.setCustomer(customer);
		customerAttributeMailingList.setCustomerOption(subscribedToMailingList);
		customerAttributeMailingList.setCustomerOptionValue(no);
		
		customer.getAttributes().add(customerAttributeMailingList);
		
		customerService.save(customer);
		
		customerService.delete(customer);
		


		
	}
}



```
