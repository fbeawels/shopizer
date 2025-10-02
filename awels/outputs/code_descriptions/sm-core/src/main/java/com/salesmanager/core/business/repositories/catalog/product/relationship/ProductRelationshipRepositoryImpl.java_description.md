# ProductRelationshipRepositoryImpl.java

## Review

## 1. Summary  

The `ProductRelationshipRepositoryImpl` implements the `ProductRelationshipRepositoryCustom` interface, providing a set of JPA/HQL queries that retrieve `ProductRelationship` entities based on a variety of filtering criteria (store, product, type, language, group, etc.).  

Key characteristics:  

* **Domain** – eCommerce product catalog.  
* **Frameworks** – Spring (via `@PersistenceContext`), JPA/Hibernate, Google Guava (`Lists`).  
* **Design** – Custom repository implementation that uses raw HQL strings rather than Spring Data’s derived queries or Criteria API.  
* **Pattern** – Data‑Access Object (DAO) with explicit query strings for complex joins and eager fetching.  

## 2. Detailed Description  

### Core components  

| Class | Responsibility |
|-------|----------------|
| `ProductRelationshipRepositoryImpl` | Executes HQL queries to fetch `ProductRelationship` objects with various filters. |
| `EntityManager` | JPA’s persistence context used for creating queries and executing them. |
| Guava `Lists` | Used to convert a `Map` of unique relationships back into a list. |

### Execution Flow  

1. **Initialization** – Spring injects an `EntityManager` into the repository.  
2. **Method call** – Client code calls one of the public methods (e.g., `getByType(store, type, product)`).  
3. **Query construction** – The method selects the appropriate HQL constant, sets parameters, and executes the query.  
4. **Result handling** – The raw list returned by `getResultList()` is returned to the caller.  
5. **Cleanup** – No explicit cleanup; the persistence context is managed by the container.

### Assumptions & Constraints  

* All query parameters (`store`, `product`, `language`, `type`) are non‑null; the code does not perform null‑checks.  
* The entity model is tightly coupled to the HQL strings – any change in entity names or relationships requires a manual update of the queries.  
* Distinct clauses are used to mitigate Cartesian product duplicates caused by the many `join fetch` clauses.  
* The repository is not thread‑safe by design – JPA’s `EntityManager` is thread‑local.

### Architectural Choices  

* **Raw HQL** – Chosen over the Criteria API or Spring Data’s method‑derived queries to provide fine‑grained control over eager fetching.  
* **`@PersistenceContext`** – Standard JPA injection, keeping the repository framework‑agnostic.  
* **Guava’s `Lists`** – Simplifies conversion from a `Map` of unique codes back to a list.  

## 3. Functions/Methods  

| Method | Signature | Purpose | Parameters | Return | Side‑Effects |
|--------|-----------|---------|------------|--------|--------------|
| `getByType(MerchantStore, String, Product, Language)` | Retrieves relationships by code (`type`), store, product ID and language. | Fetches detailed product relationships for a specific product and language. | `store`, `type`, `product`, `language` | `List<ProductRelationship>` | None |
| `getGroupByType(MerchantStore, String)` | Retrieves relationships by group code. | Returns relationships that are considered part of a group but does **not** fetch related products. | `store`, `group` | `List<ProductRelationship>` | None |
| `getByType(MerchantStore, String, Language)` | Retrieves relationships by code and language only. | Fetches all product relationships for a given type and language. | `store`, `type`, `language` | `List<ProductRelationship>` | None |
| `getByGroup(MerchantStore, String)` | Similar to `getGroupByType` but fetches related product details. | Returns fully populated relationships for a group. | `store`, `group` | `List<ProductRelationship>` | None |
| `getGroups(MerchantStore)` | Retrieves all groups for a store. | Returns distinct group codes by deduplicating on `code`. | `store` | `List<ProductRelationship>` | Uses `Map` to dedupe |
| `getByType(MerchantStore, String)` | Retrieves relationships by code only. | Returns all relationships for a given type. | `store`, `type` | `List<ProductRelationship>` | None |
| `listByProducts(Product)` | Retrieves relationships where the product or its related product matches the given ID. | Simple search by product ID. | `product` | `List<ProductRelationship>` | None |
| `getByType(MerchantStore, String, Product)` | Retrieves relationships by type, store, and product, filtering on availability. | Returns available relationships for a product. | `store`, `type`, `product` | `List<ProductRelationship>` | None |
| `getByTypeAndRelatedProduct(MerchantStore, String, Product)` | Retrieves relationships by type and *related* product ID. | Fetches relationships where the related product is the specified product and is available. | `store`, `type`, `product` | `List<ProductRelationship>` | None |

*Reusable utility:*  
`Lists.newArrayList(relationMap.values())` is a convenience to turn a `Collection` into a mutable list.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.persistence.EntityManager` | JPA (standard) | Core persistence API. |
| `javax.persistence.PersistenceContext` | JPA (standard) | Injects the `EntityManager`. |
| `com.google.api.client.util.Lists` | Guava (third‑party) | Provides a convenience `newArrayList` method. |
| `java.util.List`, `Map`, `Collectors` | JDK (standard) | Java Streams for deduplication. |
| `ProductRelationship`, `Product`, `MerchantStore`, `Language` | Application domain | Entities from the catalog domain. |

No external frameworks beyond JPA/Hibernate and Guava are required.

## 5. Additional Notes  

### Strengths  

* **Explicit eager fetching** – The use of `join fetch` ensures that the relationships are loaded in a single query, avoiding the N+1 select problem.  
* **Clear query separation** – Each query is defined once as a constant, making it easier to see the intended join structure.  
* **Deduplication logic** – `getGroups` uses a map to guarantee unique group codes.  

### Weaknesses / Risks  

1. **Duplicate method names** – Several overloads of `getByType` can be confusing and may lead to accidental misuse or IDE auto‑completion mistakes.  
2. **Hard‑coded HQL strings** – Any change to the entity model (e.g., renaming a field) will break queries without a compile‑time check.  
3. **Potential Cartesian products** – Even with `distinct`, complex `join fetch` chains can produce large result sets and heavy memory consumption.  
4. **Missing null checks** – Passing `null` for `store`, `product`, or `language` will throw a `NullPointerException`.  
5. **Redundant queries** – `getGroupByType` and `getByGroup` are very similar; the former does not fetch related product details, which might be an oversight or a deliberate performance optimization.  
6. **No paging** – All methods return full lists; for large catalogs this could lead to memory issues.  
7. **Guava dependency** – `Lists` could be replaced with `new ArrayList<>(…)` to reduce third‑party usage.  

### Suggested Enhancements  

* **Rename overloaded methods** – Use more descriptive names (`findByTypeAndProductAndLanguage`, `findByGroupCode`, etc.) to improve readability.  
* **Parameter validation** – Add defensive checks or `Objects.requireNonNull` to guard against `null` inputs.  
* **Use Criteria API or NamedQueries** – Move query logic to named queries (in the entity) or Criteria to benefit from type safety and easier maintenance.  
* **Pagination support** – Accept `Pageable` or offset/limit parameters to limit result sizes.  
* **Cache frequently used queries** – Consider Spring Cache or second‑level caching for heavy read workloads.  
* **Refactor deduplication** – The `getGroups` method could use `Collectors.toMap(..., (a,b)->a)` directly without an explicit map variable.  
* **Unit tests** – Ensure that each query returns the expected results for various edge cases (no matches, multiple matches, null fields).  

Overall, the implementation is functional and follows a clear pattern, but it would benefit from clearer method signatures, stronger type safety, and some refactoring to reduce duplication and improve maintainability.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.repositories.catalog.product.relationship;

import com.google.api.client.util.Lists;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.relationship.ProductRelationship;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.reference.language.Language;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;


public class ProductRelationshipRepositoryImpl implements ProductRelationshipRepositoryCustom {

  private static final String HQL_GET_BY_CODE_AND_STORE_ID_AND_PRODUCT_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "join fetch pr.relatedProduct rp "
          + "left join fetch rp.descriptions rpd "
          + "where pr.code=:code "
          + "and pr.store.id=:storeId "
          + "and p.id=:id "
          + "and rpd.language.id=:langId";
  private static final String HQL_GET_BY_CODE_AND_STORE_ID_AND_RP_PRODUCT_ID =
	      "select distinct pr from ProductRelationship as pr "
	          + "left join fetch pr.relatedProduct rp "
	          + "where pr.code=:code "
	          + "and pr.store.id=:storeId "
	          + "and rp.available=:available "
	          + "and rp.id=:rpid";
  private static final String HQL_GET_BY_PRODUCT_ID_AND_CODE_AVAILABLE =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "left join fetch pr.relatedProduct rp "
          + "left join fetch rp.attributes pattr "
          + "left join fetch rp.categories rpc "
          + "left join fetch rpc.descriptions rpcd "
          + "left join fetch rp.descriptions rpd "
          + "left join fetch rp.owner rpo "
          + "left join fetch rp.images pd "
          + "left join fetch rp.merchantStore rpm "
          + "left join fetch rpm.currency rpmc "
          + "left join fetch rp.availabilities pa "
          + "left join fetch pa.prices pap "
          + "left join fetch pap.descriptions papd "
          + "left join fetch rp.manufacturer manuf "
          + "left join fetch manuf.descriptions manufd "
          + "left join fetch rp.type type "
          + "where pr.code=:code "
          + "and rp.available=:available "
          + "and p.id=:pId";
  private static final String HQL_GET_PRODUCTS_BY_PRODUCT_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "left join fetch pr.relatedProduct rp "
          + "left join fetch rp.attributes pattr "
          + "left join fetch rp.categories rpc "
          + "left join fetch p.descriptions pd "
          + "left join fetch rp.descriptions rpd "
          + "where p.id=:id"
          + " or rp.id=:id";
  private static final String HQL_GET_PRODUCT_BY_CODE_AND_STORE_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "join fetch pr.relatedProduct rp "
          + "left join fetch rp.descriptions rpd "
          + "where pr.code=:code "
          + "and pr.store.id=:storeId ";
  private static final String HQL_GET_PRODUCT_BY_CODE_AND_STORE_ID_AND_LANG_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "join fetch pr.relatedProduct rp "
          + "left join fetch rp.attributes pattr "
          + "left join fetch pattr.productOption po "
          + "left join fetch po.descriptions pod "
          + "left join fetch pattr.productOptionValue pov "
          + "left join fetch pov.descriptions povd "
          + "left join fetch rp.categories rpc "
          + "left join fetch rpc.descriptions rpcd "
          + "left join fetch rp.descriptions rpd "
          + "left join fetch rp.owner rpo "
          + "left join fetch rp.images pd "
          + "left join fetch rp.merchantStore rpm "
          + "left join fetch rpm.currency rpmc "
          + "left join fetch rp.availabilities pa "
          + "left join fetch rp.manufacturer m "
          + "left join fetch m.descriptions md "
          + "left join fetch pa.prices pap "
          + "left join fetch pap.descriptions papd "
          + "where pr.code=:code "
          + "and pr.store.id=:storeId "
          + "and rpd.language.id=:langId";
  private static final String HQL_GET_GROUP_BY_CODE_AND_STORE_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "left join fetch pr.relatedProduct rp "
          + "left join fetch rp.attributes pattr "
          + "left join fetch rp.categories rpc "
          + "left join fetch rpc.descriptions rpcd "
          + "left join fetch rp.descriptions rpd "
          + "left join fetch rp.owner rpo "
          + "left join fetch rp.images pd "
          + "left join fetch rp.merchantStore rpm "
          + "left join fetch rpm.currency rpmc "
          + "left join fetch rp.availabilities pa "
          + "left join fetch pa.prices pap "
          + "left join fetch pap.descriptions papd "
          + "left join fetch rp.manufacturer manuf "
          + "left join fetch manuf.descriptions manufd "
          + "left join fetch rp.type type "
          + "where pr.code=:code "
          + "and pr.store.id=:storeId ";
  private static final String HQL_GET_PRODUCT_RELATIONSHIP_BY_STORE_ID =
      "select distinct pr from ProductRelationship as pr "
          + "where pr.store.id=:store "
          + "and pr.product=null";
  private static final String HQL_GET_PRODUCT_RELATIONSHIP_BY_CODE_AND_STORE_ID =
      "select distinct pr from ProductRelationship as pr "
          + "left join fetch pr.product p "
          + "left join fetch pr.relatedProduct rp "
          + "left join fetch rp.descriptions rpd "
          + "where pr.code=:code "
          + "and pr.store.id=:storeId "
          + "and rpd.language.id=:langId";

  @PersistenceContext
  private EntityManager entityManager;

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getByType(MerchantStore store, String type, Product product,
      Language language) {
    return entityManager.createQuery(HQL_GET_BY_CODE_AND_STORE_ID_AND_PRODUCT_ID)
        .setParameter("code", type)
        .setParameter("id", product.getId())
        .setParameter("storeId", store.getId())
        .setParameter("langId", language.getId())
        .getResultList();
  }

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getGroupByType(MerchantStore store, String type) {
    return entityManager.createQuery(HQL_GET_PRODUCT_RELATIONSHIP_BY_CODE_AND_STORE_ID)
        .setParameter("code", type)
        .setParameter("storeId", store.getId())
        .getResultList();
  }

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getByType(MerchantStore store, String type, Language language) {
    return entityManager.createQuery(HQL_GET_PRODUCT_BY_CODE_AND_STORE_ID_AND_LANG_ID)
        .setParameter("code", type)
        .setParameter("langId", language.getId())
        .setParameter("storeId", store.getId())
        .getResultList();
  }

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getByGroup(MerchantStore store, String group) {
    return entityManager.createQuery(HQL_GET_GROUP_BY_CODE_AND_STORE_ID)
        .setParameter("code", group)
        .setParameter("storeId", store.getId())
        .getResultList();
  }

  @Override
  public List<ProductRelationship> getGroups(MerchantStore store) {
    @SuppressWarnings("unchecked")
    List<ProductRelationship> relations = entityManager.createQuery(HQL_GET_PRODUCT_RELATIONSHIP_BY_STORE_ID)
        .setParameter("store", store.getId())
        .getResultList();
    Map<String, ProductRelationship> relationMap = relations.stream()
        .collect(Collectors.toMap(ProductRelationship::getCode, p -> p, (p, q) -> p));
    return Lists.newArrayList(relationMap.values());
  }


  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getByType(MerchantStore store, String type) {
    return entityManager.createQuery(HQL_GET_PRODUCT_BY_CODE_AND_STORE_ID)
        .setParameter("code", type)
        .setParameter("storeId", store.getId())
        .getResultList();
  }

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> listByProducts(Product product) {
    return entityManager.createQuery(HQL_GET_PRODUCTS_BY_PRODUCT_ID)
        .setParameter("id", product.getId())
        .getResultList();
  }

  @Override
  @SuppressWarnings("unchecked")
  public List<ProductRelationship> getByType(MerchantStore store, String type, Product product) {
    return entityManager.createQuery(HQL_GET_BY_PRODUCT_ID_AND_CODE_AVAILABLE)
        .setParameter("code", type)
        .setParameter("available", true)
        .setParameter("pId", product.getId())
        .getResultList();
  }

	@SuppressWarnings("unchecked")
	@Override
	public List<ProductRelationship> getByTypeAndRelatedProduct(MerchantStore store, String type, Product product) {
	    return entityManager.createQuery(HQL_GET_BY_CODE_AND_STORE_ID_AND_RP_PRODUCT_ID)
	            .setParameter("code", type)
	            .setParameter("available", true)
	            .setParameter("rpid", product.getId())
	            .setParameter("storeId", store.getId())
	            .getResultList();
	}
}



```
