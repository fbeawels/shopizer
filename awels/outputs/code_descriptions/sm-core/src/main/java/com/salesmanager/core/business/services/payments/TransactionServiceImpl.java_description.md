# TransactionServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`TransactionServiceImpl` is a Spring‑managed service that manages `Transaction` entities for an e‑commerce platform. It delegates persistence to a JPA/Hibernate repository (`TransactionRepository`) and implements a custom `TransactionService` interface that provides:

* Creation of transactions (with JSON detail handling).  
* Retrieval of all transactions for an order (with JSON deserialization).  
* Determination of the *last* transaction, *capturable* transaction, and *refundable* transaction for an order.  
* Listing of transactions by date range.

**Key components**

| Component | Role |
|-----------|------|
| `TransactionRepository` | DAO layer – CRUD + custom queries. |
| `TransactionService` | Service contract exposed to other layers. |
| `TransactionServiceImpl` | Concrete implementation of business rules. |
| `Transaction` | JPA entity representing a payment action. |
| `TransactionType` | Enum describing the type of transaction. |
| `ObjectMapper` | Jackson JSON parser for the `details` field. |

The implementation is straightforward, with no complex frameworks beyond Spring, JPA/Hibernate, and Jackson. No obvious design patterns other than the typical *Repository* and *Service* layers.

---

## 2. Detailed Description  

1. **Construction**  
   * The class is annotated with `@Service("transactionService")` and receives a `TransactionRepository` via JSR‑330 `@Inject`.  
   * It calls `super(transactionRepository)` to initialize the generic `SalesManagerEntityServiceImpl`.

2. **Creation** (`create(Transaction)`)  
   * Serialises the `Transaction` into JSON (`toJSONString()`) and stores that string in the `details` field if it is not blank.  
   * Delegates to the superclass for persistence.

3. **Listing Transactions for an Order** (`listTransactions(Order)`)  
   * Retrieves all transactions for the order by `orderId`.  
   * For each transaction, deserialises the `details` JSON into a `Map<String,String>` and stores it in the entity via `setTransactionDetails`.  

4. **Determining the Last Transaction** (`lastTransaction(Order, MerchantStore)`)  
   * Loads all transactions for the order.  
   * Builds a `TreeMap<String, Transaction>` keyed by `Transaction::getTransactionTypeName`.  
   * Uses `lastEntry()` to pick the “last” transaction (actually the last key in lexical order).  
   * The method currently only logs the step and returns that transaction – the logic is incomplete.

5. **Capturable Transaction** (`getCapturableTransaction(Order)`)  
   * Iterates over all transactions for the order.  
   * Looks for an `AUTHORIZE` transaction (first one encountered that is not followed by a `CAPTURE` or `REFUND`) and returns it.  

6. **Refundable Transaction** (`getRefundableTransaction(Order)`)  
   * Builds a map of the most recent `AUTHORIZECAPTURE`, `CAPTURE`, and `REFUND` transactions (by date order logic).  
   * Chooses the `CAPTURE` or `AUTHORIZECAPTURE` as the transaction that can be refunded.  
   * Deserialises the `details` JSON for the chosen transaction.

7. **Listing Transactions by Date** (`listTransactions(Date, Date)`)  
   * Delegates to the repository to fetch all transactions between two dates.

### Assumptions & Constraints  

* All operations assume the `order` and `transaction` objects are non‑null.  
* The `details` field contains a JSON representation that can always be mapped to `Map<String,String>`.  
* Ordering of transactions is performed either by key (`String`) or by internal date comparison logic.  
* No concurrency control – the service is stateless.  

### Architecture  

The class follows a classic *Service‑Repository* pattern: the service contains business logic while the repository handles persistence. No transaction boundaries are explicitly declared; they are presumably managed by the superclass or Spring’s default transaction management.  

---

## 3. Functions / Methods  

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `public void create(Transaction)` | Persists a new transaction; serialises details to JSON. | `transaction` | void | Persists transaction; sets `details` | Uses `toJSONString()` – might duplicate `details` if already set. |
| `public List<Transaction> listTransactions(Order)` | Returns all transactions for an order; deserialises details. | `order` | `List<Transaction>` | Populates `transactionDetails` for each | Creates a new `ObjectMapper` per call. |
| `public Transaction lastTransaction(Order, MerchantStore)` | Returns the last transaction of an order based on type name order. | `order`, `store` | `Transaction` | Prints current step | Logic incomplete; not date‑sorted. |
| `public Transaction getCapturableTransaction(Order)` | Finds the first `AUTHORIZE` transaction that has not been captured or refunded. | `order` | `Transaction` | None | Linear scan; may return a stale `AUTHORIZE` if multiple. |
| `public Transaction getRefundableTransaction(Order)` | Finds the most recent transaction that can be refunded. | `order` | `Transaction` | Deserialises details | Complex map logic; may override earlier captures. |
| `public List<Transaction> listTransactions(Date, Date)` | Returns all transactions between two dates. | `startDate`, `endDate` | `List<Transaction>` | None | Direct repository delegation. |

Reusable utility methods: none – all logic is inline within service methods.

---

## 4. Dependencies  

| Library / API | Role | Type |
|---------------|------|------|
| `org.springframework.stereotype.Service` | Spring bean registration | Framework |
| `javax.inject.Inject` | Constructor injection | JSR‑330 |
| `com.salesmanager.core.business.repositories.payments.TransactionRepository` | DAO layer | Project |
| `com.salesmanager.core.business.exception.ServiceException` | Exception wrapper | Project |
| `com.salesmanager.core.model.payments.Transaction` | Entity | Project |
| `com.salesmanager.core.model.payments.TransactionType` | Enum | Project |
| `org.apache.commons.lang3.StringUtils` | String utilities | Third‑party (Apache Commons Lang) |
| `com.fasterxml.jackson.databind.ObjectMapper` | JSON parsing | Third‑party (Jackson) |
| `java.util.*` | Core Java utilities | Standard |
| `java.util.stream.*` | Stream API | Standard |

No platform‑specific or heavy dependencies. The code relies on Spring’s component scanning, JPA/Hibernate (implied by repository), and Jackson for JSON handling.

---

## 5. Additional Notes & Recommendations  

### Strengths  

* Clear separation between service and repository layers.  
* Consistent use of `ServiceException` to encapsulate lower‑level errors.  
* Explicit handling of transaction details as JSON, allowing extensibility.  

### Weaknesses & Edge Cases  

1. **Ordering Logic**  
   * `lastTransaction` uses `TreeMap` keyed by `transactionTypeName`, which orders alphabetically, **not** chronologically.  
   * Capturable and refundable logic rely on manual date comparison, but may not pick the truly latest transaction if multiple of the same type exist.  

2. **Performance & Resource Usage**  
   * A new `ObjectMapper` is instantiated for every list operation – better to reuse a static instance or inject a singleton.  
   * Deserialising the entire JSON map for each transaction can be costly if many transactions are processed.  

3. **Null / Empty Checks**  
   * Methods assume non‑null arguments (`order`, `transaction`). No defensive checks.  
   * `transaction.getDetails()` may be `null` or an empty string; the current code uses `StringUtils.isBlank` which handles this, but deserialisation could still throw if the string is malformed.  

4. **Thread Safety & Transactions**  
   * No explicit transaction demarcation – rely on Spring defaults. For multi‑step updates (e.g., `create` + deserialise) an explicit `@Transactional` could be clearer.  

5. **Logging**  
   * Only `System.out.println` is used; production code should use a logger (`org.slf4j.Logger`).  

6. **Method Completeness**  
   * `lastTransaction` is incomplete – it only prints the current step and returns the last entry.  
   * The `MerchantStore` parameter is unused; either remove it or incorporate store‑specific logic.  

7. **API Design**  
   * Returning raw `List<Transaction>` that has been mutated (details set) may be surprising to callers; consider returning a DTO.  

8. **Error Handling**  
   * The catch blocks wrap generic `Exception` into `ServiceException`; more specific handling (e.g., `JsonProcessingException`) could provide clearer diagnostics.  

### Suggested Enhancements  

1. **Ordering by Date** – Replace the TreeMap approach with a query that orders by `transactionDate DESC` and returns the first result.  
2. **Repository Methods** – Add methods such as  
   ```java
   Transaction findTopByOrderIdOrderByTransactionDateDesc(Long orderId);
   Transaction findByOrderIdAndTransactionTypeNameOrderByTransactionDateDesc(Long orderId, String type);
   ```  
3. **Reuse ObjectMapper** – Inject a singleton `ObjectMapper` via Spring configuration.  
4. **Add Logging** – Replace `System.out.println` with SLF4J logging.  
5. **Null Safety** – Add checks or guard clauses for null parameters.  
6. **Unit Tests** – Implement tests covering various transaction sequences (authorize → capture → refund) to validate business rules.  
7. **DTO Layer** – Expose transaction details as a strongly typed DTO instead of raw `Map<String,String>`.  
8. **Transactional Boundaries** – Annotate public service methods with `@Transactional` where multiple DB operations may occur.

---

**Conclusion**  
The service implements the required CRUD and lookup logic for payment transactions, but its transaction‑ordering and date handling are fragile and incomplete. By refactoring the ordering logic, optimizing JSON handling, and tightening error handling and logging, the class can become more robust, maintainable, and performant.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.services.payments;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.repositories.payments.TransactionRepository;
import com.salesmanager.core.business.services.common.generic.SalesManagerEntityServiceImpl;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.order.Order;
import com.salesmanager.core.model.payments.Transaction;
import com.salesmanager.core.model.payments.TransactionType;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Service;

import javax.inject.Inject;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
import java.util.TreeMap;
import java.util.stream.Collectors;



@Service("transactionService")
public class TransactionServiceImpl  extends SalesManagerEntityServiceImpl<Long, Transaction> implements TransactionService {
	

	private TransactionRepository transactionRepository;
	
	@Inject
	public TransactionServiceImpl(TransactionRepository transactionRepository) {
		super(transactionRepository);
		this.transactionRepository = transactionRepository;
	}
	
	@Override
	public void create(Transaction transaction) throws ServiceException {
		
		//parse JSON string
		String transactionDetails = transaction.toJSONString();
		if(!StringUtils.isBlank(transactionDetails)) {
			transaction.setDetails(transactionDetails);
		}
		
		super.create(transaction);
		
		
	}
	
	@Override
	public List<Transaction> listTransactions(Order order) throws ServiceException {
		
		List<Transaction> transactions = transactionRepository.findByOrder(order.getId());
		ObjectMapper mapper = new ObjectMapper();
		for(Transaction transaction : transactions) {
				if(!StringUtils.isBlank(transaction.getDetails())) {
					try {
						@SuppressWarnings("unchecked")
						Map<String,String> objects = mapper.readValue(transaction.getDetails(), Map.class);
						transaction.setTransactionDetails(objects);
					} catch (Exception e) {
						throw new ServiceException(e);
					}
				}
		}
		
		return transactions;
	}
	
	/**
	 * Authorize
	 * AuthorizeAndCapture
	 * Capture
	 * Refund
	 * 
	 * Check transactions
	 * next transaction flow is
	 * Build map of transactions map
	 * filter get last from date
	 * get last transaction type
	 * verify which step transaction it if
	 * check if target transaction is in transaction map we are in trouble...
	 * 
	 */
	public Transaction lastTransaction(Order order, MerchantStore store) throws ServiceException {
		
		List<Transaction> transactions = transactionRepository.findByOrder(order.getId());
		//ObjectMapper mapper = new ObjectMapper();
		
		//TODO order by date
	    TreeMap<String, Transaction> map = transactions.stream()
	    	      .collect(

	    	    		  Collectors.toMap(
	    	    				  Transaction::getTransactionTypeName, transaction -> transaction,(o1, o2) -> o1, TreeMap::new)
	    	    		  
	    	    		  
	    	    		  );
	    
		  
	    
		//get last transaction
	    Entry<String,Transaction> last = map.lastEntry();
	    
	    String currentStep = last.getKey();
	    
	    System.out.println("Current step " + currentStep);
	    
	    //find next step
	    
	    return last.getValue();
	    


	}

	@Override
	public Transaction getCapturableTransaction(Order order)
			throws ServiceException {
		List<Transaction> transactions = transactionRepository.findByOrder(order.getId());
		ObjectMapper mapper = new ObjectMapper();
		Transaction capturable = null;
		for(Transaction transaction : transactions) {
			if(transaction.getTransactionType().name().equals(TransactionType.AUTHORIZE.name())) {
				if(!StringUtils.isBlank(transaction.getDetails())) {
					try {
						@SuppressWarnings("unchecked")
						Map<String,String> objects = mapper.readValue(transaction.getDetails(), Map.class);
						transaction.setTransactionDetails(objects);
						capturable = transaction;
					} catch (Exception e) {
						throw new ServiceException(e);
					}
				}
			}
			if(transaction.getTransactionType().name().equals(TransactionType.CAPTURE.name())) {
				break;
			}
			if(transaction.getTransactionType().name().equals(TransactionType.REFUND.name())) {
				break;
			}
		}
		
		return capturable;
	}
	
	@Override
	public Transaction getRefundableTransaction(Order order)
		throws ServiceException {
		List<Transaction> transactions = transactionRepository.findByOrder(order.getId());
		Map<String,Transaction> finalTransactions = new HashMap<String,Transaction>();
		Transaction finalTransaction = null;
		for(Transaction transaction : transactions) {
			if(transaction.getTransactionType().name().equals(TransactionType.AUTHORIZECAPTURE.name())) {
				finalTransactions.put(TransactionType.AUTHORIZECAPTURE.name(),transaction);
				continue;
			}
			if(transaction.getTransactionType().name().equals(TransactionType.CAPTURE.name())) {
				finalTransactions.put(TransactionType.CAPTURE.name(),transaction);
				continue;
			}
			if(transaction.getTransactionType().name().equals(TransactionType.REFUND.name())) {
				//check transaction id
				Transaction previousRefund = finalTransactions.get(TransactionType.REFUND.name());
				if(previousRefund!=null) {
					Date previousDate = previousRefund.getTransactionDate();
					Date currentDate = transaction.getTransactionDate();
					if(previousDate.before(currentDate)) {
						finalTransactions.put(TransactionType.REFUND.name(),transaction);
						continue;
					}
				} else {
					finalTransactions.put(TransactionType.REFUND.name(),transaction);
					continue;
				}
			}
		}
		
		if(finalTransactions.containsKey(TransactionType.AUTHORIZECAPTURE.name())) {
			finalTransaction = finalTransactions.get(TransactionType.AUTHORIZECAPTURE.name());
		}
		
		if(finalTransactions.containsKey(TransactionType.CAPTURE.name())) {
			finalTransaction = finalTransactions.get(TransactionType.CAPTURE.name());
		}

		if(finalTransaction!=null && !StringUtils.isBlank(finalTransaction.getDetails())) {
			try {
				ObjectMapper mapper = new ObjectMapper();
				@SuppressWarnings("unchecked")
				Map<String,String> objects = mapper.readValue(finalTransaction.getDetails(), Map.class);
				finalTransaction.setTransactionDetails(objects);
			} catch (Exception e) {
				throw new ServiceException(e);
			}
		}
		
		return finalTransaction;
	}

	@Override
	public List<Transaction> listTransactions(Date startDate, Date endDate) throws ServiceException {
		
		return transactionRepository.findByDates(startDate, endDate);
	}

}



```
