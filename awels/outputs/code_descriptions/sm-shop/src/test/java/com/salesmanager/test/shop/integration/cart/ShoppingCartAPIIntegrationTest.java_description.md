# ShoppingCartAPIIntegrationTest.java

## Review

## 1. Summary  

The file is a JUnit 5 integration test for a Spring‑Boot based shopping‑cart REST API.  
* **Purpose** – Verify that the cart endpoints behave correctly for a series of typical user interactions (add, update, delete, error handling).  
* **Key components**  
  * `@SpringBootTest` boots the full application context on a random port.  
  * `TestRestTemplate` drives HTTP requests to the exposed endpoints.  
  * `CartTestBean` holds shared state (cart id, products) across ordered tests.  
  * Each test method is annotated with `@Order` to enforce execution sequence.  
* **Notable patterns/frameworks** – Spring MVC for the API, Spring Test framework for integration testing, Hamcrest for assertions.

---

## 2. Detailed Description  

### Core Flow  

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1. **Add first item** | POST `/api/v1/cart/` with product SKU & qty 1 | 201 CREATED, cart created, `quantity == 1` |
| 2. **Add second item** | PUT `/api/v1/cart/{id}` | 201 CREATED, cart now holds 2 items (`quantity == 2`) |
| 3. **Bad cart id** | PUT `/api/v1/cart/{badId}` | 404 NOT_FOUND |
| 4. **Bulk update** | POST `/api/v1/cart/{id}/multi` | 201 CREATED, quantities updated, total `quantity == 4` |
| 5. **Zero quantity update** | POST `/api/v1/cart/{id}/multi` with qty 0 | 201 CREATED, item removed, `quantity == 2` |
| 6. **Delete item** | DELETE `/api/v1/cart/{id}/product/{productId}` | 204 NO_CONTENT, body null |
| 7. **Delete with body** | DELETE `/api/v1/cart/{id}/product/{sku}?body=true` | 200 OK, remaining cart returned |

The test harness relies on the fact that all methods execute in a fixed order, sharing the same cart id and product list via the static `data` bean. After each test the state is either asserted or mutated (e.g., product removed).

### Assumptions & Constraints  

* The application is expected to run on a fresh instance for each test run – no persistence beyond the lifetime of the test process.  
* `sampleProduct(String)` – a helper that presumably creates or fetches a product; its implementation is not shown but is assumed to succeed.  
* `getHeader()` – builds the HTTP headers (likely authentication).  
* The test uses a **static** shared state (`data`), which couples tests tightly; any failure earlier in the sequence can cascade and make later tests unreliable.  

### Design Choices  

* **Ordered tests**: Simplifies cart lifecycle management but sacrifices isolation; a failure in one test can invalidate the rest.  
* **REST calls via `TestRestTemplate`**: Provides end‑to‑end coverage, but the test does not validate the body of responses beyond a few fields.  
* **Hard‑coded URLs**: String concatenation rather than `UriComponentsBuilder`; minor readability issue but acceptable in tests.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `addToCart()` | Create new cart with one item. | None | None | Sets `data.cartId` |
| `addSecondToCart()` | Append another item to the existing cart. | None | None | |
| `addToWrongToCartId()` | Test error handling for non‑existent cart id. | None | None | Removes added product from `data.products` |
| `updateMultiWCartId()` | Bulk update of two items’ quantities. | None | None | |
| `updateMultiWZeroOnOneProd()` | Bulk update with quantity 0 to remove an item. | None | None | |
| `deleteCartItem()` | Delete a specific cart item by product id, expect no content. | None | None | |
| `deleteCartItemWithBody()` | Delete an item but request the updated cart in the body. | None | None | |

All test methods are void and rely on assertions for validation. They share state via the static `data` bean.

---

## 4. Dependencies  

| Library | Type | Usage |
|---------|------|-------|
| `spring-boot-starter-test` | Third‑party | Provides JUnit 5, Spring Test, MockMvc, TestRestTemplate |
| `spring-boot-starter-web` | Third‑party | REST controller, `WebEnvironment.RANDOM_PORT` |
| `hamcrest-core` | Third‑party | `assertThat` with matchers |
| `spring-test` | Third‑party | `TestRestTemplate`, `SpringExtension` |
| JUnit 5 (`@Test`, `@Order`, etc.) | Third‑party | Test lifecycle |
| **Application code** – `ShopApplication`, DTOs (`ReadableProduct`, `PersistableShoppingCartItem`, `ReadableShoppingCart`), helper `ServicesTestSupport` – all internal. |

No platform‑specific dependencies; the test runs on any JVM with the above libraries.

---

## 5. Additional Notes & Recommendations  

### Strengths  

* Tests cover a complete user flow, including error handling.  
* Using a random port prevents port conflicts in CI environments.  
* Clear use of HTTP status assertions (CREATED, NOT_FOUND, NO_CONTENT, OK).  

### Potential Issues  

1. **Coupling & Order Dependence**  
   * The reliance on `@Order` and shared static state makes the test suite fragile. If `addToCart()` fails, all subsequent tests will likely fail or behave unpredictably.  
   * Recommended: Refactor to use `@BeforeEach` to create a fresh cart for each test, or use a dedicated test fixture that cleans up after each test.  

2. **Lack of Cleanup**  
   * No explicit deletion of the cart after tests. While the in‑memory context is destroyed after the test run, a production‑grade system with persistent storage would need cleanup to avoid data leakage.  

3. **Hard‑coded Assertions**  
   * The test assumes specific quantity totals (`2`, `4`, `5`, `2`). While accurate for the current sequence, any change to the cart logic will require manual updates to these numbers.  
   * Consider deriving expected values programmatically (e.g., sum of product quantities) to reduce maintenance overhead.  

4. **Missing Body Validation**  
   * For most responses only the status code and `quantity` are verified. It would be beneficial to assert the full structure (e.g., cart code, items list).  

5. **Error Path Coverage**  
   * Only one error case is tested (`404` on bad cart id). Additional negative tests (e.g., invalid payload, authentication failures) could improve coverage.  

6. **Magic Strings & URLs**  
   * Use `UriComponentsBuilder` to build URLs; this reduces errors and improves readability.  

7. **Thread Safety**  
   * The static `data` bean is accessed by all tests on the same thread; however, if tests run in parallel (JUnit 5 allows parallel execution), shared state would cause race conditions. Ensure tests are configured to run sequentially or remove static shared state.  

### Future Enhancements  

* **Parameterized Tests** – Use JUnit 5's `@ParameterizedTest` to run the same flow with different product sets.  
* **Spring Data Test Support** – If the cart is persisted, use an embedded database to verify data consistency after each operation.  
* **Mock Security** – Replace real authentication with `@WithMockUser` or similar to avoid external dependencies.  
* **DTO Validation** – Add tests that assert DTO field constraints (e.g., mandatory SKU, non‑negative quantity).  
* **Performance Checks** – Measure response times for bulk operations.  

--- 

**Overall,** the integration test is functional and covers the basic success path and a few error conditions. With minor refactoring to decouple tests and add deeper validation, it can become a robust, maintainable part of the test suite.

## Code Critique



## Code Preview

```java
package com.salesmanager.test.shop.integration.cart;

import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertThat;
import static org.springframework.http.HttpStatus.CREATED;
import static org.springframework.http.HttpStatus.NOT_FOUND;
import static org.springframework.http.HttpStatus.NO_CONTENT;
import static org.springframework.http.HttpStatus.OK;

import org.junit.jupiter.api.MethodOrderer;
import org.junit.jupiter.api.Order;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestMethodOrder;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.salesmanager.shop.application.ShopApplication;
import com.salesmanager.shop.model.catalog.product.ReadableProduct;
import com.salesmanager.shop.model.shoppingcart.PersistableShoppingCartItem;
import com.salesmanager.shop.model.shoppingcart.ReadableShoppingCart;
import com.salesmanager.test.shop.common.ServicesTestSupport;

@SpringBootTest(classes = ShopApplication.class, webEnvironment = WebEnvironment.RANDOM_PORT)
@ExtendWith(SpringExtension.class)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class ShoppingCartAPIIntegrationTest extends ServicesTestSupport {

    @Autowired
    private TestRestTemplate testRestTemplate;

    private static CartTestBean data = new CartTestBean();


    /**
     * Add an Item & Create cart, would give HTTP 201 & 1 qty
     *
     * @throws Exception
     */
    @Test
    @Order(1)
    public void addToCart() throws Exception {

    	ReadableProduct product = sampleProduct("addToCart");
    	assertNotNull(product);
    	data.getProducts().add(product);

        PersistableShoppingCartItem cartItem = new PersistableShoppingCartItem();
        cartItem.setProduct(product.getSku());
        cartItem.setQuantity(1);

        final HttpEntity<PersistableShoppingCartItem> cartEntity = new HttpEntity<>(cartItem, getHeader());
        final ResponseEntity<ReadableShoppingCart> response = testRestTemplate.postForEntity(String.format("/api/v1/cart/"), cartEntity, ReadableShoppingCart.class);

        data.setCartId(response.getBody().getCode());

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(CREATED));
        assertEquals(response.getBody().getQuantity(), 1);

    }

    /**
     * Add an second Item to existing Cart, should give HTTP 201 & 2 qty
     *
     * @throws Exception
     */
    @Test
    @Order(2)
    public void addSecondToCart() throws Exception {

        ReadableProduct product = sampleProduct("add2Cart2");
        assertNotNull(product);
        data.getProducts().add(product);

        PersistableShoppingCartItem cartItem = new PersistableShoppingCartItem();
        cartItem.setProduct(product.getSku());
        cartItem.setQuantity(1);

        final HttpEntity<PersistableShoppingCartItem> cartEntity = new HttpEntity<>(cartItem, getHeader());
        final ResponseEntity<ReadableShoppingCart> response = testRestTemplate.exchange(String.format("/api/v1/cart/" + String.valueOf(data.getCartId())),
                HttpMethod.PUT,
                cartEntity,
                ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(CREATED));
        assertEquals(response.getBody().getQuantity(), 2);
    }

    /**
     * Add an other item to cart which then does not exist which should return HTTP 404
     *
     * @throws Exception
     */
    @Test
    @Order(3)
    public void addToWrongToCartId() throws Exception {

        ReadableProduct product = sampleProduct("add3Cart");
        assertNotNull(product);
        data.getProducts().add(product);

        PersistableShoppingCartItem cartItem = new PersistableShoppingCartItem();
        cartItem.setProduct(product.getSku());
        cartItem.setQuantity(1);

        final HttpEntity<PersistableShoppingCartItem> cartEntity = new HttpEntity<>(cartItem, getHeader());
        final ResponseEntity<ReadableShoppingCart> response = testRestTemplate.exchange(String.format("/api/v1/cart/" + data.getCartId() + "breakIt"),
                HttpMethod.PUT,
                cartEntity,
                ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(NOT_FOUND));
        data.getProducts().remove(product);
    }


    /**
     * Update cart items with qty 2 (1) on existing items & adding new item with qty 1 which gives result 2x2+1 = 5
     *
     * @throws Exception
     */
    @Test
    @Order(4)
    public void updateMultiWCartId() throws Exception {

        PersistableShoppingCartItem cartItem1 = new PersistableShoppingCartItem();
        cartItem1.setProduct(data.getProducts().get(0).getSku());
        cartItem1.setQuantity(2);

        PersistableShoppingCartItem cartItem2 = new PersistableShoppingCartItem();
        cartItem2.setProduct(data.getProducts().get(1).getSku());
        cartItem2.setQuantity(2);


        PersistableShoppingCartItem[] productsQtyUpdates = {cartItem1, cartItem2};


        final HttpEntity<PersistableShoppingCartItem[]> cartEntity = new HttpEntity<>(productsQtyUpdates, getHeader());
        final ResponseEntity<ReadableShoppingCart> response = testRestTemplate.exchange(String.format("/api/v1/cart/" + data.getCartId() +
                        "/multi"),
                HttpMethod.POST,
                cartEntity,
                ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(CREATED));
        assertEquals(4, response.getBody().getQuantity());
    }

    /**
     * Update cart with qnt 0 on one cart item which gives 3 qty left
     *
     * @throws Exception
     */
    @Test
    @Order(5)
    public void updateMultiWZeroOnOneProd() throws Exception {

        PersistableShoppingCartItem cartItem1 = new PersistableShoppingCartItem();
        cartItem1.setProduct(data.getProducts().get(0).getSku());
        cartItem1.setQuantity(0);

        PersistableShoppingCartItem[] productsQtyUpdates = {cartItem1};


        final HttpEntity<PersistableShoppingCartItem[]> cartEntity = new HttpEntity<>(productsQtyUpdates, getHeader());
        final ResponseEntity<ReadableShoppingCart> response = testRestTemplate.exchange(String.format("/api/v1/cart/" + data.getCartId() +
                        "/multi"),
                HttpMethod.POST,
                cartEntity,
                ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(CREATED));
        assertEquals(2, response.getBody().getQuantity());
    }

    /**
     * Delete cartitem from cart, should return 204 / no content
     *
     * @throws Exception
     */
    @Test
    @Order(6)
    public void deleteCartItem() throws Exception {

        final ResponseEntity<ReadableShoppingCart> response =
                testRestTemplate.exchange(String.format("/api/v1/cart/" + data.getCartId() + "/product/" + String.valueOf(data.getProducts().get(1).getId())),
                HttpMethod.DELETE,
                null,
                ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(NO_CONTENT));
        assertNull(response.getBody());
    }

    /**
     * Delete cartitem from cart with body set to true which gives remaining cart content, should return 200 / ok
     *
     * @throws Exception
     */
    @Test
    @Order(7)
    public void deleteCartItemWithBody() throws Exception {

        final ResponseEntity<ReadableShoppingCart> response =
                testRestTemplate.exchange(String.format("/api/v1/cart/" + data.getCartId() + "/product/" + String.valueOf(data.getProducts().get(1).getSku()) + "?body=true"),
                        HttpMethod.DELETE,
                        null,
                        ReadableShoppingCart.class);

        assertNotNull(response);
        assertThat(response.getStatusCode(), is(OK));
    }

}



```
