# User.java

## Review

## 1. Summary  
**Purpose**  
`User` is a JPA entity that represents an administrative user of a merchant store in the SalesManager system. It holds authentication data, personal details, security questions, audit metadata, and the groups that the user belongs to.  

**Key Components**  
| Component | Role |
|-----------|------|
| `@Entity` / `@Table` | Maps the class to the `USERS` database table with a composite unique key on (`MERCHANT_ID`,`ADMIN_NAME`). |
| `@Id`, `@GeneratedValue` | Primary key generation via a table generator. |
| `@ManyToMany` (groups) | Associates a user with multiple `Group` entities through the `USER_GROUP` join table. |
| `@ManyToOne` (merchantStore / defaultLanguage) | Links the user to a `MerchantStore` and optionally a default `Language`. |
| `@Embedded` (`AuditSection`, `CredentialsReset`) | Embeds audit and password‑reset information into the same table. |
| `@EntityListeners(AuditListener.class)` | Hooks into the entity lifecycle for automatic audit field population. |
| Validation annotations (`@NotEmpty`, `@Email`) | Enforces basic data integrity at runtime. |

**Design Patterns / Frameworks**  
* JPA/Hibernate for ORM.  
* Hibernate’s `@Cascade` (for fine‑grained cascade control).  
* Validation API for declarative constraints.  
* Audit pattern via `Auditable` interface and `AuditListener`.

---

## 2. Detailed Description  

### Architecture  
The class extends `SalesManagerEntity<Long, User>` (a generic base entity that likely implements `Serializable` and common fields) and implements `Auditable`. It is a pure persistence model with no business logic; all behaviour is delegated to services/controllers.  

### Execution Flow  
1. **Persistence** – When persisted, Hibernate creates or updates a row in `USERS`.  
2. **Audit** – `AuditListener` populates `AuditSection` fields (`createdBy`, `createdOn`, `updatedBy`, `updatedOn`) before insert/update.  
3. **Password Handling** – The class simply stores a password string; hashing/validation is expected to happen elsewhere (e.g., in a service layer).  
4. **Group Association** – The `groups` list is lazily loaded; fetching groups requires an additional database hit unless configured otherwise.  
5. **Cleanup** – No explicit cleanup logic; garbage collection handles object lifecycle.

### Assumptions & Constraints  
* `adminName` must be unique per `MerchantStore`.  
* Passwords are expected to be hashed before persisting.  
* `adminEmail` is validated as an email format but not necessarily unique.  
* The class is designed for relational databases that support JPA tables and sequences.  

### Design Choices  
* **Table generator** is used for portability across DBMS that lack native sequence support.  
* **Lazy loading** for many‑to‑many groups keeps the default entity lightweight.  
* **Embedded audit** keeps audit columns co‑located with entity columns, simplifying queries.  
* **Validation annotations** keep data integrity in one place but require the calling layer to trigger validation.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `User()` | No‑arg constructor required by JPA. | – | – | – |
| `User(String userName, String password, String email)` | Convenience ctor. | `userName`, `password`, `email` | – | Sets fields `adminName`, `adminPassword`, `adminEmail`. |
| `getId()/setId(Long)` | JPA id accessors (from `SalesManagerEntity`). | – | – | – |
| `getAuditSection()/setAuditSection(AuditSection)` | Accessor for audit data. | – | – | – |
| `getAdminName()/setAdminName(String)` | User name. | – | – | – |
| `getAdminEmail()/setAdminEmail(String)` | Email. | – | – | – |
| `getAdminPassword()/setAdminPassword(String)` | Password. | – | – | – |
| `getFirstName()/setFirstName(String)` | First name. | – | – | – |
| `getLastName()/setLastName(String)` | Last name. | – | – | – |
| `getDefaultLanguage()/setDefaultLanguage(Language)` | Preferred language. | – | – | – |
| `getQuestion1()/setQuestion1(String)` | Security question 1. | – | – | – |
| `getQuestion2()/setQuestion2(String)` | Security question 2. | – | – | – |
| `getQuestion3()/setQuestion3(String)` | Security question 3. | – | – | – |
| `getAnswer1()/setAnswer1(String)` | Answer 1. | – | – | – |
| `getAnswer2()/setAnswer2(String)` | Answer 2. | – | – | – |
| `getAnswer3()/setAnswer3(String)` | Answer 3. | – | – | – |
| `getGroups()/setGroups(List<Group>)` | User‑group associations. | – | – | – |
| `getMerchantStore()/setMerchantStore(MerchantStore)` | Link to merchant. | – | – | – |
| `isActive()/setActive(boolean)` | Activation flag. | – | – | – |
| `getLastAccess()/setLastAccess(Date)` | Last activity timestamp. | – | – | – |
| `getLoginTime()/setLoginTime(Date)` | Login timestamp. | – | – | – |
| `getCredentialsResetRequest()/setCredentialsResetRequest(CredentialsReset)` | Password reset details. | – | – | – |

**Reusable/Utility Methods** – None. All are simple POJO accessors.

---

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| `javax.persistence` | JPA (core) | Entity mapping, lifecycle. |
| `javax.validation` | Bean Validation | Declarative field constraints. |
| `org.hibernate.annotations` | Hibernate | Advanced cascade, table generator. |
| `com.salesmanager.core.model.common.audit.*` | Internal | Audit listener and section. |
| `com.salesmanager.core.model.common.CredentialsReset` | Internal | Embedded reset token holder. |
| `com.salesmanager.core.model.generic.SalesManagerEntity` | Internal | Base entity (likely implements `Serializable`). |
| `com.salesmanager.core.model.merchant.MerchantStore` | Internal | Many‑to‑one relation. |
| `com.salesmanager.core.model.reference.language.Language` | Internal | Many‑to‑one relation. |
| `com.salesmanager.core.model.user.Group` | Internal | Many‑to‑many relation. |

All dependencies are either standard JPA/validation APIs or project‑specific. No platform‑specific or exotic libraries are used.

---

## 5. Additional Notes  

### Security Concerns  
* **Plain‑text password** – The field stores whatever is passed in. Hashing must be performed **before** setting `adminPassword`.  
* **Security questions** – Storing plain answers is risky; consider hashing or removing this approach.  

### Validation & Business Logic  
* Validation annotations are present, but the class does not enforce them automatically. Service layer must invoke a validator or use a JPA provider that triggers validation on persist/update.  
* No `equals()` / `hashCode()` overrides – this can cause issues when entities are used in collections or as keys. It is advisable to implement these methods based on the immutable `id` or business key (`adminName`).  

### Performance  
* `groups` is lazy and may trigger N+1 selects if accessed in a loop. Consider fetching strategies or DTO projections for common use‑cases.  
* The composite unique constraint ensures no duplicate usernames per merchant, but may impact performance under high concurrency.

### Future Enhancements  
1. **Add `equals()` / `hashCode()`** (based on `id` or a business key).  
2. **Implement `toString()`** that omits sensitive fields.  
3. **Expose a DTO** for API layers to avoid leaking internal fields.  
4. **Use a dedicated password hashing service** to encapsulate the algorithm (e.g., BCrypt).  
5. **Encrypt security answers** if they must be retained.  
6. **Audit fields** could be automatically populated using JPA lifecycle callbacks (`@PrePersist`, `@PreUpdate`) instead of a separate listener.  
7. **Validate email uniqueness** via a unique constraint if required.  

### Edge Cases  
* **Null language** – `defaultLanguage` is nullable; ensure code handles missing language gracefully.  
* **Transient vs. persistent groups** – adding a new `Group` instance without persisting it first may cause `TransientObjectException`.  
* **Multiple merchants** – The unique constraint is on `MERCHANT_ID` + `ADMIN_NAME`; attempting to copy a user across merchants will require updating the name.

Overall, the class is a clean JPA mapping that delegates business concerns to other layers. Adding the suggested robustness improvements will make it safer and easier to maintain.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.model.user;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;

import javax.persistence.CascadeType;
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.EntityListeners;
import javax.persistence.FetchType;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Index;
import javax.persistence.JoinColumn;
import javax.persistence.JoinTable;
import javax.persistence.ManyToMany;
import javax.persistence.ManyToOne;
import javax.persistence.Table;
import javax.persistence.TableGenerator;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.persistence.UniqueConstraint;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotEmpty;

import org.hibernate.annotations.Cascade;

import com.salesmanager.core.model.common.CredentialsReset;
import com.salesmanager.core.model.common.audit.AuditListener;
import com.salesmanager.core.model.common.audit.AuditSection;
import com.salesmanager.core.model.common.audit.Auditable;
import com.salesmanager.core.model.generic.SalesManagerEntity;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;

/**
 * User management
 * @author carlsamson
 *
 */
@Entity
@EntityListeners(value = AuditListener.class)
@Table(name = "USERS", 
    indexes = { @Index(name="USR_NAME_IDX", columnList = "ADMIN_NAME")},
	uniqueConstraints=
	@UniqueConstraint(columnNames = {"MERCHANT_ID", "ADMIN_NAME"}))
public class User extends SalesManagerEntity<Long, User> implements Auditable {
	
	
	private static final long serialVersionUID = 1L;
	
	@Id
	@Column(name = "USER_ID", unique=true, nullable=false)
	@TableGenerator(name = "TABLE_GEN", table = "SM_SEQUENCER", pkColumnName = "SEQ_NAME", valueColumnName = "SEQ_COUNT", pkColumnValue = "USER_SEQ_NEXT_VAL")
	@GeneratedValue(strategy = GenerationType.TABLE, generator = "TABLE_GEN")
	private Long id;
	
	public User() {
		
	}
	
	public User(String userName,String password, String email) {
		
		this.adminName = userName;
		this.adminPassword = password;
		this.adminEmail = email;
	}
	
	@NotEmpty
	@Column(name="ADMIN_NAME", length=100)
	private String adminName;
	
	@ManyToMany(fetch=FetchType.LAZY, cascade = {CascadeType.REFRESH})
	@JoinTable(name = "USER_GROUP", joinColumns = { 
			@JoinColumn(name = "USER_ID", nullable = false, updatable = false) }
			, 
			inverseJoinColumns = { @JoinColumn(name = "GROUP_ID", 
					nullable = false, updatable = false) }
	)
	@Cascade({
		org.hibernate.annotations.CascadeType.DETACH,
		org.hibernate.annotations.CascadeType.LOCK,
		org.hibernate.annotations.CascadeType.REFRESH,
		org.hibernate.annotations.CascadeType.REPLICATE
		
	})
	private List<Group> groups = new ArrayList<Group>();
	
	@NotEmpty
	@Email
	@Column(name="ADMIN_EMAIL")
	private String adminEmail;
	
	@NotEmpty
	@Column(name="ADMIN_PASSWORD", length=60)
	private String adminPassword;
	
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name="MERCHANT_ID", nullable=false)
	private MerchantStore merchantStore;
	
	
	@Column(name="ADMIN_FIRST_NAME")
	private String firstName;
	
	@Column(name="ACTIVE")
	private boolean active = true;
	
	
	@Column(name="ADMIN_LAST_NAME")
	private String lastName;
	
	@ManyToOne(fetch = FetchType.LAZY, targetEntity = Language.class)
	@JoinColumn(name = "LANGUAGE_ID")
	private Language defaultLanguage;
	
	
	@Column(name="ADMIN_Q1")
	private String question1;
	
	@Column(name="ADMIN_Q2")
	private String question2;
	
	@Column(name="ADMIN_Q3")
	private String question3;
	
	@Column(name="ADMIN_A1")
	private String answer1;
	
	@Column(name="ADMIN_A2")
	private String answer2;
	
	@Column(name="ADMIN_A3")
	private String answer3;

	@Embedded
	private AuditSection auditSection = new AuditSection();
	
	@Temporal(TemporalType.TIMESTAMP)
	@Column(name = "LAST_ACCESS")
	private Date lastAccess;
	
	@Temporal(TemporalType.TIMESTAMP)
	@Column(name = "LOGIN_ACCESS")
	private Date loginTime;
	
	@Embedded
	private CredentialsReset credentialsResetRequest = null;


	public CredentialsReset getCredentialsResetRequest() {
		return credentialsResetRequest;
	}

	public void setCredentialsResetRequest(CredentialsReset credentialsResetRequest) {
		this.credentialsResetRequest = credentialsResetRequest;
	}

	@Override
	public Long getId() {
		return this.id;
	}

	@Override
	public void setId(Long id) {
		this.id = id;
	}

	@Override
	public AuditSection getAuditSection() {
		return auditSection;
	}

	@Override
	public void setAuditSection(AuditSection audit) {
		auditSection = audit;
		
	}

	public String getAdminName() {
		return adminName;
	}

	public void setAdminName(String adminName) {
		this.adminName = adminName;
	}

	public String getAdminEmail() {
		return adminEmail;
	}

	public void setAdminEmail(String adminEmail) {
		this.adminEmail = adminEmail;
	}

	public String getAdminPassword() {
		return adminPassword;
	}

	public void setAdminPassword(String adminPassword) {
		this.adminPassword = adminPassword;
	}

	public String getFirstName() {
		return firstName;
	}

	public void setFirstName(String firstName) {
		this.firstName = firstName;
	}

	public String getLastName() {
		return lastName;
	}

	public void setLastName(String lastName) {
		this.lastName = lastName;
	}

	public Language getDefaultLanguage() {
		return defaultLanguage;
	}

	public void setDefaultLanguage(Language defaultLanguage) {
		this.defaultLanguage = defaultLanguage;
	}

	public String getQuestion1() {
		return question1;
	}

	public void setQuestion1(String question1) {
		this.question1 = question1;
	}

	public String getQuestion2() {
		return question2;
	}

	public void setQuestion2(String question2) {
		this.question2 = question2;
	}

	public String getQuestion3() {
		return question3;
	}

	public void setQuestion3(String question3) {
		this.question3 = question3;
	}

	public String getAnswer1() {
		return answer1;
	}

	public void setAnswer1(String answer1) {
		this.answer1 = answer1;
	}

	public String getAnswer2() {
		return answer2;
	}

	public void setAnswer2(String answer2) {
		this.answer2 = answer2;
	}

	public String getAnswer3() {
		return answer3;
	}

	public void setAnswer3(String answer3) {
		this.answer3 = answer3;
	}

	public void setGroups(List<Group> groups) {
		this.groups = groups;
	}

	public List<Group> getGroups() {
		return groups;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}

	public void setActive(boolean active) {
		this.active = active;
	}

	public boolean isActive() {
		return active;
	}

	public void setLastAccess(Date lastAccess) {
		this.lastAccess = lastAccess;
	}

	public Date getLastAccess() {
		return lastAccess;
	}

	public void setLoginTime(Date loginTime) {
		this.loginTime = loginTime;
	}

	public Date getLoginTime() {
		return loginTime;
	}

/*	public String getResetPasswordToken() {
		return resetPasswordToken;
	}

	public void setResetPasswordToken(String resetPasswordToken) {
		this.resetPasswordToken = resetPasswordToken;
	}

	public Date getTokenPasswordExpiration() {
		return tokenPasswordExpiration;
	}

	public void setTokenPasswordExpiration(Date tokenPasswordExpiration) {
		this.tokenPasswordExpiration = tokenPasswordExpiration;
	}*/
	

}



```
