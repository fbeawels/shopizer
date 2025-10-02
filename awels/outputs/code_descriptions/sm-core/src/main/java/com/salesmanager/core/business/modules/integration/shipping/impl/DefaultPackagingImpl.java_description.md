# DefaultPackagingImpl.java

## Review

## 1. Summary  

`DefaultPackagingImpl` is a concrete implementation of the `Packaging` interface that calculates the physical packaging (box or item‑by‑item) required for a list of products that are about to be shipped.  
* **Key responsibilities**  
  * Resolve shipping configuration for a merchant (box size & weight limits).  
  * Convert a list of `ShippingProduct` objects into either a set of box dimensions (optimised packing) or a flat list of item dimensions (per‑item packing).  
  * Persist diagnostic information to `MerchantLogService` when a product cannot be accommodated.  

* **Design notes**  
  * Uses **dependency injection** (`@Inject`) for the two service dependencies.  
  * Relies on a fairly naïve packing algorithm that simply iterates through products, fits them into existing boxes when possible, or creates a new box.  
  * Contains an internal, non‑static helper class `PackingBox` that tracks the remaining volume/weight of a box.

## 2. Detailed Description  

### 2.1 Flow of execution  

| Phase | What happens | Notes |
|-------|--------------|-------|
| **Initial validation** | `products` list is checked for `null`. The shipping configuration is retrieved and verified to exist. | Throws `ServiceException` if missing. |
| **Dimensions extraction** | Shipping box dimensions (`width`, `length`, `height`, `weight`, `maxWeight`) are taken from `ShippingConfiguration`. | If any dimension is `0`, a `MerchantLog` is created and a `ServiceException` is thrown. |
| **Product flattening** | Each `ShippingProduct` is expanded into one or more `Product` instances, each representing a single unit.  Virtual products are skipped. | Attributes and descriptions are copied.  Default dimensions are used when the product has `null` values. |
| **Packing** | An iterative loop processes every individual product: <br> 1. Validate that the product dimensions and weight are within the box limits. <br> 2. Attempt to place the product in an existing box if the remaining volume (≥ 75 % of the product volume) and weight allow. <br> 3. If not, create a new `PackingBox`. | *Bug*: the 75 % check is applied but the remaining volume is never updated after a product is placed. <br> *Bug*: `maxBox` is decremented but never used to stop packing. |
| **PackageDetails creation** | For each box, a `PackageDetails` instance is built using the merchant store’s code, the box dimensions and the cumulative weight (base weight + box’s current weight). | *Bug*: `details.setShippingWeight(weight + box.getWeight())` uses the local variable `box` (the last created box) instead of the current `pb`. |
| **Return** | A `List<PackageDetails>` is returned. | If no real products were found, the method returns `null` (instead of an empty list). |

`getItemPackagesDetails` follows a similar pattern but simply creates one `PackageDetails` per item (or per quantity unit) without attempting to group them into boxes.

### 2.2 Assumptions & Constraints  

* The shipping configuration must be present for the given store; otherwise an exception is thrown.  
* The product dimensions/weights are stored as `BigDecimal` but are converted to `double` for packing calculations.  
* Box limits are enforced strictly; any product that exceeds the limits triggers a log and exception.  
* The algorithm assumes a single box type (fixed dimensions) and does not consider multiple box sizes or optimal packing strategies.  
* The implementation is not thread‑safe – state is stored in local collections only.

### 2.3 Architectural notes  

* **Service‑centric** – the logic is encapsulated in a single service class that orchestrates other services.  
* **Procedural packing** – the packing algorithm is implemented inline, making it difficult to unit‑test or replace.  
* **Logging** – uses a custom `MerchantLogService` for error reporting but falls back to `System.out.println` for debug output.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `getBoxPackagesDetails(List<ShippingProduct>, MerchantStore)` | Calculates optimal box packing for a list of products. | `products` – list of shipping products, `store` – merchant store | `List<PackageDetails>` – one entry per box | Persists logs on errors, may throw `ServiceException`. |
| `getItemPackagesDetails(List<ShippingProduct>, MerchantStore)` | Produces one package detail per physical item. | Same as above | `List<PackageDetails>` – one per unit | Persists logs on errors, may throw `ServiceException`. |
| `PackingBox` (inner class) | Helper DTO to hold remaining volume, weight, and accumulated weight for a box. | None | N/A | None |

*Reusable/utility methods* – None; all logic is inlined within the two public methods.

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `com.salesmanager.core.business.services.shipping.ShippingService` | Service | Retrieves shipping configuration. |
| `com.salesmanager.core.business.services.system.MerchantLogService` | Service | Persists diagnostic logs. |
| `com.salesmanager.core.model.*` | Domain model | Product, Attribute, PackageDetails, ShippingConfiguration, MerchantStore, MerchantLog, etc. |
| Java SE (`java.math.BigDecimal`, `java.util.*`) | Standard | Core language utilities. |
| CDI / JSR‑330 (`javax.inject.Inject`) | Dependency Injection | Not framework‑specific but relies on a container that supports CDI. |

All dependencies are third‑party domain classes or standard Java APIs; no external libraries (e.g., Apache Commons, Guava) are used.

## 5. Additional Notes & Recommendations  

### 5.1 Bug Fixes  
1. **Shipping weight calculation** – replace `details.setShippingWeight(weight + box.getWeight())` with `details.setShippingWeight(weight + pb.getWeight())`.  
2. **Volume update** – after placing a product in a box, subtract the product volume from `pbox.setVolumeLeft(volumeLeft - productVolume)`.  
3. **Box limit enforcement** – use `maxBox` to prevent creation of more than 100 boxes or return a meaningful error.  
4. **Return empty list** – when `iterCount == 0`, return `Collections.emptyList()` instead of `null`.  
5. **Logging** – replace `System.out.println` with a proper logger (e.g., SLF4J).  

### 5.2 Design Improvements  
* **Separate packing logic** – extract the packing algorithm into a dedicated class or strategy (`BoxPacker`) to simplify testing and allow alternate algorithms (first‑fit, best‑fit, etc.).  
* **Avoid product duplication** – instead of creating a new `Product` object per quantity, work with a lightweight DTO that holds only the needed dimensions and quantity.  
* **Use `BigDecimal` for all calculations** – preserve precision, especially for weight/volume; convert to `double` only for the final `PackageDetails`.  
* **Configuration validation** – validate the shipping configuration once and cache the values rather than retrieving them for each call.  
* **Error handling** – rather than throwing a generic `ServiceException` on any dimension/weight violation, return a result object that indicates failure with an explanatory message.  

### 5.3 Performance & Scalability  
* The current algorithm is O(n²) in the worst case (each product potentially examined against all existing boxes). For large orders this may become expensive.  
* The constant `maxBox` is hard‑coded; consider making it configurable or deriving it from the store’s shipping policy.  

### 5.4 Testing & Documentation  
* No unit tests are present. Tests should cover: normal packing, edge cases (zero dimensions, oversized products), and error paths.  
* Javadoc comments for the public methods and the `PackingBox` class would aid future maintainers.  

### 5.5 Code Quality  
* Rename constants from `defaultWeight` etc. to `DEFAULT_WEIGHT`.  
* Replace `Double` primitives with `double` to avoid unnecessary autoboxing.  
* Mark the helper class `PackingBox` as `private static` if it does not capture any outer state.  

Implementing these changes will make the packaging logic more robust, maintainable, and easier to test.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.business.modules.integration.shipping.impl;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

import javax.inject.Inject;

import com.salesmanager.core.business.exception.ServiceException;
import com.salesmanager.core.business.services.shipping.ShippingService;
import com.salesmanager.core.business.services.system.MerchantLogService;
import com.salesmanager.core.model.catalog.product.Product;
import com.salesmanager.core.model.catalog.product.attribute.ProductAttribute;
import com.salesmanager.core.model.merchant.MerchantStore;
import com.salesmanager.core.model.shipping.PackageDetails;
import com.salesmanager.core.model.shipping.ShippingConfiguration;
import com.salesmanager.core.model.shipping.ShippingProduct;
import com.salesmanager.core.model.system.MerchantLog;
import com.salesmanager.core.modules.integration.shipping.model.Packaging;

public class DefaultPackagingImpl implements Packaging {

	
	@Inject
	private ShippingService shippingService;
	
	@Inject
	private MerchantLogService merchantLogService;
	
	/** default dimensions **/
	private final static Double defaultWeight = 1D;
	private final static Double defaultHeight = 4D;
	private final static Double defaultLength = 4D;
	private final static Double defaultWidth = 4D;
	
	@Override
	public List<PackageDetails> getBoxPackagesDetails(
			List<ShippingProduct> products, MerchantStore store)
			throws ServiceException {

		
		if (products == null) {
			throw new ServiceException("Product list cannot be null !!");
		}

		double width = 0;
		double length = 0;
		double height = 0;
		double weight = 0;
		double maxweight = 0;

		//int treshold = 0;
		
		
		ShippingConfiguration shippingConfiguration = shippingService.getShippingConfiguration(store);
		if(shippingConfiguration==null) {
			throw new ServiceException("ShippingConfiguration not found for merchant " + store.getCode());
		}
		
		width = (double) shippingConfiguration.getBoxWidth();
		length = (double) shippingConfiguration.getBoxLength();
		height = (double) shippingConfiguration.getBoxHeight();
		weight = shippingConfiguration.getBoxWeight();
		maxweight = shippingConfiguration.getMaxWeight();
		


		List<PackageDetails> boxes = new ArrayList<PackageDetails>();

		// maximum number of boxes
		int maxBox = 100;
		int iterCount = 0;

		List<Product> individualProducts = new ArrayList<Product>();

		// need to put items individually
		for(ShippingProduct shippingProduct : products){

			Product product = shippingProduct.getProduct();
			if (product.isProductVirtual()) {
				continue;
			}

			int qty = shippingProduct.getQuantity();

			Set<ProductAttribute> attrs = shippingProduct.getProduct().getAttributes();

			// set attributes values
			BigDecimal w = product.getProductWeight();
			BigDecimal h = product.getProductHeight();
			BigDecimal l = product.getProductLength();
			BigDecimal wd = product.getProductWidth();
			if(w==null) {
				w = new BigDecimal(defaultWeight);
			}
			if(h==null) {
				h = new BigDecimal(defaultHeight);
			}
			if(l==null) {
				l = new BigDecimal(defaultLength);
			}
			if(wd==null) {
				wd = new BigDecimal(defaultWidth);
			}
			if (attrs != null && attrs.size() > 0) {
				for(ProductAttribute attribute : attrs) {
					if(attribute.getProductAttributeWeight()!=null) {
						w = w.add(attribute.getProductAttributeWeight());
					}
				}
			}
			


			if (qty > 1) {

				for (int i = 1; i <= qty; i++) {
					Product temp = new Product();
					temp.setProductHeight(h);
					temp.setProductLength(l);
					temp.setProductWidth(wd);
					temp.setProductWeight(w);
					temp.setAttributes(product.getAttributes());
					temp.setDescriptions(product.getDescriptions());
					individualProducts.add(temp);
				}
			} else {
				Product temp = new Product();
				temp.setProductHeight(h);
				temp.setProductLength(l);
				temp.setProductWidth(wd);
				temp.setProductWeight(w);
				temp.setAttributes(product.getAttributes());
				temp.setDescriptions(product.getDescriptions());
				individualProducts.add(temp);
			}
			iterCount++;
		}

		if (iterCount == 0) {
			return null;
		}

		int productCount = individualProducts.size();

		List<PackingBox> boxesList = new ArrayList<PackingBox>();

		//start the creation of boxes
		PackingBox box = new PackingBox();
		// set box max volume
		double maxVolume = width * length * height;

		if (maxVolume == 0 || maxweight == 0) {
			
			merchantLogService.save(new MerchantLog(store,"shipping","Check shipping box configuration, it has a volume of "
							+ maxVolume + " and a maximum weight of "
							+ maxweight
							+ ". Those values must be greater than 0."));
			
			throw new ServiceException("Product configuration exceeds box configuraton");
			

		}
		
		
		box.setVolumeLeft(maxVolume);
		box.setWeightLeft(maxweight);

		boxesList.add(box);//assign first box

		//int boxCount = 1;
		List<Product> assignedProducts = new ArrayList<Product>();

		// calculate the volume for the next object
		if (assignedProducts.size() > 0) {
			individualProducts.removeAll(assignedProducts);
			assignedProducts = new ArrayList<Product>();
		}

		boolean productAssigned = false;

		for(Product p : individualProducts) {

			//Set<ProductAttribute> attributes = p.getAttributes();
			productAssigned = false;

			double productWeight = p.getProductWeight().doubleValue();


			// validate if product fits in the box
			if (p.getProductWidth().doubleValue() > width
					|| p.getProductHeight().doubleValue() > height
					|| p.getProductLength().doubleValue() > length) {
				// log message to customer
				merchantLogService.save(new MerchantLog(store,"shipping","Product "
						+ p.getSku()
						+ " has a demension larger than the box size specified. Will use per item calculation."));
				throw new ServiceException("Product configuration exceeds box configuraton");

			}

			if (productWeight > maxweight) {
				merchantLogService.save(new MerchantLog(store,"shipping","Product "
						+ p.getSku()
						+ " has a weight larger than the box maximum weight specified. Will use per item calculation."));
				
				throw new ServiceException("Product configuration exceeds box configuraton");

			}

			double productVolume = (p.getProductWidth().doubleValue()
					* p.getProductHeight().doubleValue() * p
					.getProductLength().doubleValue());

			if (productVolume == 0) {
				
				merchantLogService.save(new MerchantLog(store,"shipping","Product "
						+ p.getSku()
						+ " has one of the dimension set to 0 and therefore cannot calculate the volume"));
				
				throw new ServiceException("Product configuration exceeds box configuraton");
				

			}
			
			if (productVolume > maxVolume) {
				
				throw new ServiceException("Product configuration exceeds box configuraton");
				
			}

			//List boxesList = boxesList;

			// try each box
			//Iterator boxIter = boxesList.iterator();
			for (PackingBox pbox : boxesList) {
				double volumeLeft = pbox.getVolumeLeft();
				double weightLeft = pbox.getWeightLeft();

				if ((volumeLeft * .75) >= productVolume
						&& pbox.getWeightLeft() >= productWeight) {// fit the item
																	// in this
																	// box
					// fit in the current box
					volumeLeft = volumeLeft - productVolume;
					pbox.setVolumeLeft(volumeLeft);
					weightLeft = weightLeft - productWeight;
					pbox.setWeightLeft(weightLeft);

					assignedProducts.add(p);
					productCount--;

					double w = pbox.getWeight();
					w = w + productWeight;
					pbox.setWeight(w);
					productAssigned = true;
					maxBox--;
					break;

				}

			}

			if (!productAssigned) {// create a new box

				box = new PackingBox();
				// set box max volume
				box.setVolumeLeft(maxVolume);
				box.setWeightLeft(maxweight);

				boxesList.add(box);

				double volumeLeft = box.getVolumeLeft() - productVolume;
				box.setVolumeLeft(volumeLeft);
				double weightLeft = box.getWeightLeft() - productWeight;
				box.setWeightLeft(weightLeft);
				assignedProducts.add(p);
				productCount--;
				double w = box.getWeight();
				w = w + productWeight;
				box.setWeight(w);
				maxBox--;
			}

		}

		// now prepare the shipping info

		// number of boxes

		//Iterator ubIt = usedBoxesList.iterator();

		System.out.println("###################################");
		System.out.println("Number of boxes " + boxesList.size());
		System.out.println("###################################");

		for(PackingBox pb : boxesList) {
			PackageDetails details = new PackageDetails();
			details.setShippingHeight(height);
			details.setShippingLength(length);
			details.setShippingWeight(weight + box.getWeight());
			details.setShippingWidth(width);
			details.setItemName(store.getCode());
			boxes.add(details);
		}

		return boxes;

	}

	@Override
	public List<PackageDetails> getItemPackagesDetails(
			List<ShippingProduct> products, MerchantStore store)
			throws ServiceException {
		
		
		List<PackageDetails> packages = new ArrayList<PackageDetails>();
		for(ShippingProduct shippingProduct : products) {
			Product product = shippingProduct.getProduct();

			if (product.isProductVirtual()) {
				continue;
			}

			//BigDecimal weight = product.getProductWeight();
			Set<ProductAttribute> attributes = product.getAttributes();
			// set attributes values
			BigDecimal w = product.getProductWeight();
			BigDecimal h = product.getProductHeight();
			BigDecimal l = product.getProductLength();
			BigDecimal wd = product.getProductWidth();
			if(w==null) {
				w = new BigDecimal(defaultWeight);
			}
			if(h==null) {
				h = new BigDecimal(defaultHeight);
			}
			if(l==null) {
				l = new BigDecimal(defaultLength);
			}
			if(wd==null) {
				wd = new BigDecimal(defaultWidth);
			}
			if (attributes != null && attributes.size() > 0) {
				for(ProductAttribute attribute : attributes) {
					if(attribute.getAttributeAdditionalWeight()!=null && attribute.getProductAttributeWeight() !=null) {
						w = w.add(attribute.getProductAttributeWeight());
					}
				}
			}
			
			

			if (shippingProduct.getQuantity() == 1) {
				PackageDetails detail = new PackageDetails();

	
				detail.setShippingHeight(h
						.doubleValue());
				detail.setShippingLength(l
						.doubleValue());
				detail.setShippingWeight(w.doubleValue());
				detail.setShippingWidth(wd.doubleValue());
				detail.setShippingQuantity(shippingProduct.getQuantity());
				String description = "item";
				if(product.getDescriptions().size()>0) {
					description = product.getDescriptions().iterator().next().getName();
				}
				detail.setItemName(description);
	
				packages.add(detail);
			} else if (shippingProduct.getQuantity() > 1) {
				for (int i = 0; i < shippingProduct.getQuantity(); i++) {
					PackageDetails detail = new PackageDetails();
					detail.setShippingHeight(h
							.doubleValue());
					detail.setShippingLength(l
							.doubleValue());
					detail.setShippingWeight(w.doubleValue());
					detail.setShippingWidth(wd
							.doubleValue());
					detail.setShippingQuantity(1);//issue seperate shipping
					String description = "item";
					if(product.getDescriptions().size()>0) {
						description = product.getDescriptions().iterator().next().getName();
					}
					detail.setItemName(description);
					
					packages.add(detail);
				}
			}
		}
		
		return packages;
		
		
		
	}


}


class PackingBox {

	private double volumeLeft;
	private double weightLeft;
	private double weight;

	public double getVolumeLeft() {
		return volumeLeft;
	}

	public void setVolumeLeft(double volumeLeft) {
		this.volumeLeft = volumeLeft;
	}

	public double getWeight() {
		return weight;
	}

	public void setWeight(double weight) {
		this.weight = weight;
	}

	public double getWeightLeft() {
		return weightLeft;
	}

	public void setWeightLeft(double weightLeft) {
		this.weightLeft = weightLeft;
	}

}




```
