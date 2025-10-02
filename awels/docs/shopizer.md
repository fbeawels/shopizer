## Website Content: https://shopizerecommerce.github.io/documentation

## Converted on 2025-10-02 06:59:56 Total pages: 29

## Table of Contents

| 1. Home                                       |
|-----------------------------------------------|
| 2. api/auth/index.html                        |
| 3. api/examples/index.html                    |
| 4. api/examples/product-with-variants         |
| 5. api/products/index.html                    |
| 6. api/products/product-brands.html           |
| 7. api/products/product-category.html         |
| 8. api/products/product-definition.html       |
| 9. api/products/product-images.html           |
| 10. api/products/product-instances-group.html |
| 11. api/products/product-instances.html       |
| 12. api/products/product-listing.html         |
| 13. api/products/product-options.html         |
| 14. api/products/product-properties.html      |
| 15. api/products/product-search.html          |
| 16. api/products/product-types.html           |
| 17. api/products/product-variants.html        |
| 18. api/products/product.html                 |
| 19. architecture/index.html                   |
| 20. build-and-run.html                        |
| 21. configuration/database.html               |
| 22. configuration/emails.html                 |
| 23. docker.html                               |
| 25. model/merchant.html                       |

26. model/product-references.html
27. model/user.html
28. tools/translation.html
29. what-is-shopizer.html

## 1. Home

URL: https://shopizer-ecommerce.github.io/documentation

## Shopizer

![Image 1](shopizer_image_001.jpeg)

## Open source Java headless e-commerce

## News

- Tailwind CSS / Tailwing components demo on the way
- We have a shop react app so you can test the software
- Moving towards service based Headless commerce
- We taking on new projects every semester
- Terraform automation scripts

## Community

Join Shopizer Community on Slack

## https://communityinviter.com/apps/shopizer/shopizer

If you want to be part of the community check the Contributing part

## 2. api/auth/index.html

URL: https://shopizer-ecommerce.github.io/documentation/api/auth/index.html

## Authentication API Reference

## Administration authentication

Administration API (/private) require authentication. Authentication is performed through an endpoint accepting administration username and password. Subsequent calls to all private api require the presence of a Bearer token in the request header.

## Method:

POST

## Url:

/api/v1/private/login

## Body:

```
js { "username":"admin@shopizer.com", "password":"password" }
```

## Response:

status: OK

code: 200

js

```
{ "id": 1, "token": "...eyJhbGciOiJIUzUxMiJ9....." }
```

Private (admin) api subsequent requests require the token to be sent in the request header sh

```
curl --location --request GET 'http://localhost:8080/api/v1/private/product --header 'Authorization: Bearer ...eyJhbGciOiJIUzUxMiJ9.....' \ --header 'Content-Type: application/json'
```

## Swagger documentations

All apis are described on Swaggerhub Shopizer repository

Shopize Swagger Documentationhttps://app.swaggerhub.com/apis-docs/shopizer/shopizer-restapi/3.0.1

## 3. api/examples/index.html

URL: https://shopizer-ecommerce.github.io/documentation/api/examples/index.html

## Product with variants

Products can be configured with characteristics. Characteristics are defined outside of the product and can used during the configuration of different products.

- Product with variants

## 4. api/examples/product-with-variants

URL: https://shopizer-ecommerce.github.io/documentation/api/examples/product-with-variants

## Product Configuration Example

HEADPHONES

![Image 2](shopizer_image_002.jpeg)

An eaxample for the creation of a product with variants

## Create Brand

## Description:

API for creating a manufacturer / Brand

## Method:

POST

## Url:

/api/v1/private/manufacturer

## Body:

js

```
{ "code" : "simbeats", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "SimBeats" }, { "language" : "fr", "name" : "SimBeats" } ] }
```

## Response: status: Created code: 201 js

```
{ "id": 50, "code": "simbeats", "order": 1, "descriptions": [ { "id": 0, "language": "en", "name": "SimBeats", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null }, { "id": 0, "language": "fr", "name": "SimBeats", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } ] }
```

## Create Category

## Description:

Creates a category

## Method:

POST

Url:

## Body:

```
js { "code" : "headsets", "sortOrder" : 0, "visible" : true, "parent" : null, "descriptions" : [ { "language" : "en", "name" : "Headsets", "friendlyUrl" : "headsets", "title" : "Headsets" }, { "language" : "fr", "name" : "Écouteurs", "friendlyUrl" : "ecouteurs", "title" : "Écouteurs" } ] }
```

## Response:

status: Created

code: 201

js

```
{ "id": 51, "code": "headsets", "sortOrder": 0, "visible": true, "featured": false, "lineage": null, "depth": 0, "parent": null, "descriptions": [ { "language": "en", "name": "Headsets", "description": null, "friendlyUrl": "headsets", "keyWords": null, "highlights": null, "metaDescription": null, "title": "SimBeats Headsets" }, { "language": "fr", "name": "Écouteurs", "description": null, "friendlyUrl": "ecouteurs", "keyWords": null, "highlights": null, "metaDescription": null, "title": "SimBeats Écouteurs" } ], "children": [] }
```

## Create Product Definition SIMBEATS

## Description:

Product definition can also be immediately attached to category by adding an array of category id in which the product will be created.

"categories": [ { "id": 51 } ],

## Method:

POST

## Url:

/api/v2/private/product?lang=en&amp;store=DEFAULT

## Body:

js

```
{ "canBePurchased": true, "price": 149.00, "quantity": 1000, "descriptions": [ { "description": "The preferred choice of a vast range of acclaimed DJs "friendlyUrl": "simbeatsep", "name": "SIMBeats EP", "title": "SIMBeats EP", "language": "en" }, { "description": "Le choix préféré des audiophiles, fréquences des bass "friendlyUrl": "simbeatsep", "name": "SIMBeats EP", "title": "SIMBeats EP", "language": "fr" } ], "identifier": "SIMBEATSEP", "productSpecifications": { "dimensionUnitOfMeasure": "in", "height": 8, "length": 4, "weight": 0.225, "weightUnitOfMeasure": "lb", "manufacturer": "simbeats", "width": 12 }, "categories" : [ { "code":"simbeats"} ], "shipeable": true, "sortOrder": 1, "type": "GENERAL", "virtual": false, "visible": true }
```

## Response:

## status: Created

## code: 201

```
js { "id": 26050 }
```

## Edit product SIMBEATS

## Description:

Edit product definition. This is usefull to update the details of a product.

## Method:

PUT

## Url:

/api/v2/private/product/1

## Body:

js

```
{ "canBePurchased": true, "price": 149.00, "quantity": 1000, "descriptions": [ { "description": "The preferred choice of a vast range of acclaimed DJs "friendlyUrl": "simbeatsep", "name": "SIMBeats EP", "title": "SIMBeats EP", "language": "en" }, { "description": "Le choix préféré des audiophiles, fréquences des bass "friendlyUrl": "simbeatsep", "name": "SIMBeats EP", "title": "SIMBeats EP", "language": "fr" } ], "identifier": "SIMBEATSEP", "productSpecifications": { "dimensionUnitOfMeasure": "in", "height": 8, "length": 4, "weight": 0.225, "weightUnitOfMeasure": "lb", "manufacturer": "simbeats", "width": 12 }, "categories" : [ { "code":"headsets"} ], "shipeable": true, "sortOrder": 1, "type": "GENERAL", "virtual": false, "visible": true }
```

## Response:

## status: OK

code: 200

js

## Create Product Option Color

## Description:

API for the creation of a product option

## Method:

POST

## Url:

/api/v1/private/product/option

## Body:

```
js { "code": "color", "descriptions": [ { "name": "Color", "language": "en" }, { "name": "Couleur", "language": "fr" } ], "order": 1, "type": "radio" }
```

## Response:

status: Created

## code: 201

```
js { "id": 1, "code": "color", "type": "radio", "readOnly": false, "order": 0, "description": { "id": 1, "language": "en", "name": "Color", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value Color - Red

## Description:

API for the creation of a product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "red", "descriptions": [ { "name": "Red", }, { } ], "order": 0
```

```
"language": "en" "name": "Rouge", "language": "fr" } Response: status: Created code: 201 js
```

```
{ "id": 1, "code": "red", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 2, "language": "en", "name": "Red", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value Color - Blue

## Description:

API for the creation of a product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "blue", "descriptions": [ { "name": "Blue", "language": "en" }, { "name": "Bleu", "language": "fr" } ], "order": 5 }
```

## Response: status: Created code: 201 js

```
{ "id": 2, "code": "blue", "name": null, "defaultValue": false, "sortOrder": 5, "image": null, "order": 5, "price": null, "description": { "id": 4, "language": "en", "name": "Blue", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value Color - Black

## Description:

API for the creation of a product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "black", "descriptions": [ { }, { "name": "Noir", } ], "order": 10
```

```
"name": "Black", "language": "en" "language": "fr" } Response: status: Created code: 201 js
```

```
{ "id": 3, "code": "black", "name": null, "defaultValue": false, "sortOrder": 10, "image": null, "order": 10, "price": null, "description": { "id": 6, "language": "en", "name": "Black", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create Product Option Cable

## Description:

API for the creation of a product option

## Method:

POST

## Url:

/api/v1/private/product/option

## Body:

js

```
{ "code": "cable", "descriptions": [ { }, { } ], "order": 1, "type": "radio"
```

```
"name": "Cable", "language": "en" "name": "Cable", "language": "fr" } Response: status: Created code: 201 js { "id": 3, "code": "cable", "type": "radio", "readOnly": false, "order": 0, "description": { "id": 5, "language": "en", "name": "Cable", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value Straight

## Description:

API for the creation of a product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "straight", "descriptions": [ { "name": "Straight", "language": "en" }, { "name": "Droit", "language": "fr" } ], "order": 0 }
```

## Response:

status: Created

code: 201

js

```
{ "id": 1, "code": "red", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 2, "language": "en", "name": "Red", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Response:

status: Created

code: 201

js

```
{ "id": 4, "code": "straight", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 7, "language": "en", "name": "Straight", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value Twisted

## Description:

API for the creation of a product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "twisted", "descriptions": [ { "name": "Twisted", "language": "en" }, { "language": "fr" } ], "order": 0
```

## "name": "Tortillé", } Response: status: Created code: 201 js

```
{ "id": 5, "code": "twisted", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 10, "language": "en", "name": "Twisted", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create Product Variation- Color Red

## Description:

Creates a Product variation that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variation

## Body:

js

```
{ "code": "color-red", "defaultValue": true, "option": 1, "optionValue": 1, "sortOrder": 0 } Response: status: Created code: 201 js { "id": 0 }
```

## Create Product Variation - Color Blue

## Description:

Creates a Product variation that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variation

## Body:

js

```
{ "code": "color-blue", "defaultValue": true, "option": 1, "optionValue": 2, "sortOrder": 30 } Response: status: Created code: 201 js { "id": 0 }
```

## Create Product Variation - Color Black

## Description:

Creates a Product variation that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variation

## Body:

js

```
{ "code": "color-black", "defaultValue": false, "option": 1, "optionValue": 3, "sortOrder": 30 } Response: status: Created code: 201 js { "id": 0 }
```

## Create Product Variation - Cable Straight

## Description:

Creates a Product variation that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variation

## Body:

js

```
{ "code": "cable-straight", "defaultValue": true, "option": 2, "optionValue": 50, "sortOrder": 0 } Response: status: Created code: 201 js { "id": 0 }
```

## Create Product Variation - Cable Twisted

## Description:

Creates a Product variation that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variation

## Body:

js

```
{ "code": "cable-twisted", "defaultValue": false, "option": 3, "optionValue": 5, "sortOrder": 10 } Response: status: Created code: 201 js { "id": 0 }
```

## Create Product Variant - Red - Straight

## Description:

Creates a product variant

## Method:

POST

## Url:

/api/v2/private/product/1/variant

## Body:

js

```
{ "visible": true, "sortOrder": 1, "sku" : "SIMBEATSEP001", "variation": 1, "variationValue": 3
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js { "id": 1 }
```

## Create Product Variant - Red - Twisted

## Description:

Creates a product variant

## Method:

POST

## Url:

/api/v2/private/product/1/variant

## Body:

js

```
{ "visible": true, "sortOrder": 10, "sku" : "SIMBEATSEP010", "variation": 1, "variationValue": 4
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js { "id": 2 }
```

## Create Product Variant - Black - Straight

## Description:

Creates a product instance

## Method:

POST

## Url:

/api/v2/private/product/1/variant

## Body:

js

```
{ "visible": true, "sortOrder": 1, "sku" : "SIMBEATSEP020", "variation": 2, "variationValue": 3
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js { "id": 1 }
```

## Create Product Variant - Black - Twisted

## Description:

Creates a product instance

## Method:

POST

## Url:

/api/v2/private/product/1/variant

## Body:

js

```
{ "visible": true, "sortOrder": 1, "sku" : "SIMBEATSEP024", "variation": 2, "variationValue": 4
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js { "id": 1 }
```

## Create Product Variant - Blue - Straight

## Description:

Creates a product instance

## Method:

POST

## Url:

/api/v2/private/product/1/variant

## Body:

js

```
{ "visible": true, "sortOrder": 1, "sku" : "SIMBEATSEP020", "variation": 2, "variationValue": 3
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js { "id": 1 } Create Product Variant - Blue - Twisted Description: Creates a product instance Method: POST Url: /api/v2/private/product/1/variant Body:
```

js

```
{ "visible": true, "sortOrder": 1, "sku" : "SIMBEATSEP035", "variation": 50, "variationValue": 4
```

```
{ }
```

```
"dateAvailable": "2022-11-11", } Response: status: Created code: 201 js "id": 1
```

## get product

## Description:

Get a product by sku.

## Method:

GET

## Url:

/api/v2/product/SIMBEATSEP?lang=en&amp;store=DEFAULT

## Response:

status: OK

code: 200

```
{ "id": 25655, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1172, "quantity": 3, "sku": "79702984044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "79702984044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25244, "language": "en", "name": "Wrap-around handguard kit", "description": "<ul><li>Including all required assembly materials</ "friendlyUrl": "wrap-around-handguard-kit", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Wrap-around handguard kit" }, "productPrice": {
```

```
"id": 50558, "originalPrice": "SEK1,172.00", "finalPrice": "SEK1,172.00", "discounted": false, "description": { "id": 50372, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,172.00", "originalPrice": "SEK1,172.00", "discounted": false, "image": null, "images": [ { "id": 2103, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFAULT/7970 "externalUrl": "https://cms.jinomotos.com/pim/7/7b75de5b-c86b-4 "videoUrl": "https://cms.jinomotos.com/pim/7/7b75de5b-c86b-4ba8 "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2104, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFAULT/7970 "externalUrl": "https://cms.jinomotos.com/pim/2/285a6966-77b2-4 "videoUrl": "https://cms.jinomotos.com/pim/2/285a6966-77b2-43ae "imageType": 1, "order": 0, "defaultImage": false } ],
```

```
"manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346, "language": "en", "name": "Hand Guards", "description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true,
```

## 5. api/products/index.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/index.html

## Product API Reference

## Product characteristics

Products can be configured with characteristics. Characteristics are defined outside of the product and can used during the configuration of different products.

- Product types
- Product brands

## Product options

Products can have options. Options are used to differenciate products having the same definition. For instance, a product color or a product size are options of the same base product. Options are defined outside of the product and can used during the configuration of different products.

- Product options

## Product definition

Product definition is product metadata that defines core non variables elements of a product. For instance a t-shirt is a basic product definition. A product definition has no variants attached. It contains basic characteristiques, core attributes and dynamic attributes.

- Product definition
- Product images

## Product

Product definition returns product summary usefull when working on a product management perspective. Product api returns complete object with composition rules and should be used when listing products or getting product details.

- Product

## Product properties (attributes)

Properties (attributes) adds extension to the product core attributes by allowing the creation of custom properties (attributes).

- Product custom properties

## Product variations (options)

Variations of product definitions.

- Product variations

## Product variants

Product variants are product definitions with variations attached to it. A single product definition can have multiple product variants. Each product variant has its own sku.

- Product variants
- Product variants group

## Product classification

Products are classified in Category.

- Product category

## List products

List products with pagination and filters for a specific Merchant Store and Language.

- Product listing

## Search products

Full Text search and Auto complete keywords using Search server such as Elastic Search and Open Search

- Product search

## Swagger documentations

All apis are described on Swaggerhub Shopizer repository Shopize Swagger Documentationhttps://app.swaggerhub.com/apis-docs/shopizer/shopizer-rest-

## api/3.0.1

## 6. api/products/product-brands.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-brands.html

## Product Brands

An API that can be used for managing product brands, manufacturers or collections.

## Create Product Prand / Manufacturer

## Description:

API for creating a manufacturer / Brand

## Method:

POST

## Url:

## Body:

```
js { "code" : "gilden-like", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "Gilden Like" }, { "language" : "fr", "name" : "Gilden Like" } ] }
```

## Response:

status: Created

code: 201

js

```
{ "id": 50, "code": "gilden-like", "order": 1, "descriptions": [ { "id": 0, "language": "en", "name": "Gilden Like", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null }, { "id": 0, "language": "fr", "name": "Gilden Like", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } ] }
```

## Update Product Brand/ Manufacturer

## Description:

API for updating a brand

## Method:

PUT

Url:

## Body:

```
js { "order": 10, "code": "gilden-like", "descriptions": [ { "language": "en", "name": "Gilden Like" }, { "language": "fr", "name": "De Gilden" } ] } Response: status: OK code: 200 js
```

## List Product Brands / Manufacturers

## Description:

API for updating a brand

## Method:

GET

## Url:

/api/v1/manufacturers?count=25&amp;page=0

## Response: status: OK code: 200 js

```
{ "totalPages": 1, "number": 0, "recordsTotal": 2, "recordsFiltered": 0, "manufacturers": [ { "id": 1, "code": "DEFAULT", "order": 0, "description": { "id": 1, "language": "en", "name": "DEFAULT", "description": "DEFAULT", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, { "id": 50, "code": "gilden-like", "order": 0, "description": { "id": 2, "language": "en", "name": "Gilden Like", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } } ] }
```

## Get Product Brand / Manufacturer

## Description:

```
Get a Brand / Manufacturer Method: GET Url: /api/v1/manufacturer/1 Response: status: OK code: 200 js { "id": 1, "code": "DEFAULT", "order": 0, "description": { "id": 1, "language": "en", "name": "DEFAULT", "description": "DEFAULT", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Delete Product Brand / Manufacturer

Description:

## Method:

DELETE

## Url:

/api/v1/private/manufacturer/50

## Response: status: OK code: 200 js

## 7. api/products/product-category.html

## URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-category.html

## Product Category

Create a category of products. Items (product definitions) can be associated to multiple category. Product instances cannot be associated to a category. Only the product definition can.

## Create Category

## Description:

Creates a category

## Method:

POST

## Url:

## Body:

```
{ "code" : "cotton-shirts", "sortOrder" : 0, "visible" : true, "parent" : null, "descriptions" : [ { "language" : "en", "name" : "Cotton Shirts", "friendlyUrl" : "cotton-shirts", "title" : "Standard cotton t-shirts" }]
```

```
js } Response: status: Created code: 201 js
```

```
{ "id": 10651, "code": "cotton-shirts", "sortOrder": 0, "visible": true, "featured": false, "lineage": null, "depth": 0, "parent": null, "descriptions": [ { "id": 0, "language": "en", "name": "Cotton Shirts", "description": null, "friendlyUrl": "cotton-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Standard cotton t-shirts" } ], "children": [] }
```

## Modify category

## Description:

Modify a Category

## Method:

PUT

## Url:

/api/v1/private/category/10651

## Body:

```
{ "code" : "cotton-shirts", "sortOrder" : 1, "visible" : true, "parent" : null, "descriptions" : [ { "language" : "en", "name" : "Cotton Shirts", "friendlyUrl" : "cotton-shirts", "title" : "Standard cotton t-shirts" }]
```

```
} Response: status: OK code: 200 js
```

```
{ "id": 10651, "code": "cotton-shirts", "sortOrder": 1, "visible": true, "featured": false, "lineage": null, "depth": 0, "parent": null, "descriptions": [ { "id": 0, "language": "en", "name": "Cotton Shirts", "description": null, "friendlyUrl": "cotton-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Standard cotton t-shirts" } ], "children": [] }
```

## Get Category

## Description:

Modify a Category

## Method:

GET

## Url:

/api/v1/category/10651

## Body:

```
{ "code" : "cotton-shirts", "sortOrder" : 1, "visible" : true, "parent" : null, "descriptions" : [ { "language" : "en", "name" : "Cotton Shirts", "friendlyUrl" : "cotton-shirts", "title" : "Standard cotton t-shirts" }]
```

```
} Response: status: OK code: 200 js
```

```
{ "id": 10700, "code": "cotton-shirts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10700/", "depth": 0, "parent": null, "description": { "id": 371, "language": "en", "name": "Cotton Shirts", "description": null, "friendlyUrl": "cotton-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Standard cotton t-shirts" }, "productCount": 0, "store": "DEFAULT", "children": [] }
```

## List Category

## Description:

Get a category hierarchy for a given MerchantStore

## Method:

GET

## Url:

/api/v1/category?store-DEFAULT

## Body:

```
{ "code" : "cotton-shirts", "sortOrder" : 1, "visible" : true, "parent" : null, "descriptions" : [ { "language" : "en", "name" : "Cotton Shirts", "friendlyUrl" : "cotton-shirts", "title" : "Standard cotton t-shirts" }]
```

```
} Response: status: OK code: 200 js
```

```
{ "id": 10700, "code": "cotton-shirts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10700/", "depth": 0, "parent": null, "description": { "id": 371, "language": "en", "name": "Cotton Shirts", "description": null, "friendlyUrl": "cotton-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Standard cotton t-shirts" }, "productCount": 0, "store": "DEFAULT", "children": [] }
```

## Add product to category

## Description:

Add a product to a category

This url adds product ID 251 to category 100 Product and category must have the same merchant store

## Method:

POST

## Url:

## Response: status: Created code: 201 js

## Remove product from category

## Description:

Add a product to a category This url adds product ID 251 to category 100 Product and category must have the same merchant store

## Method:

POST

## Url:

/api/v1/private/product/26650/category/10651

## Response: status: OK code: 200 js

## Delete Category

## Description:

Delete a Category

## Method:

DELETE

## Url:

/api/v1/private/category/10651

## Response:

status: OK

code: 200

js

## 8. api/products/product-definition.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-definition.html

## Product Definition

Product definitions represent product metadata that can be used for multiple items (sku). For instance a product definition t-shirt is the metadata for small white t-shirt and medium white t-shirt.

## create product definition

## Description:

Product definition can also be immediately attached to category by adding an array of category id in which the product will be created.

```
"categories": [ { "id": 51 } ],
```

## Method:

POST

## Url:

/api/v2/private/product?lang=en&amp;store=DEFAULT

## Body:

```
{ "canBePurchased": true, "price": 29.00, "quantity": 100, "descriptions": [ { "description": "T Shirt", "friendlyUrl": "t-shirt", "name": "T Shirt", "title": "T Shirt", "language": "en" } ], "identifier": "TSHIRT", "productSpecifications": { "dimensionUnitOfMeasure": "in", "height": 22, "length": 14, "weight": 0.225, "weightUnitOfMeasure": "lb", "manufacturer": "DEFAULT", "width": 36 }, "shipeable": true, "sortOrder": 10, "type": "GENERAL", "virtual": false, "visible": true } Response: status: Created code: 201
```

js

```
{ "id": 26050 }
```

## edit product definition

## Description:

Edit product definition

## Method:

PUT

## Url:

/api/v2/private/product/26750

## Body:

js

```
"description": "Light cotton T Shirt", "friendlyUrl": "light-cotton-t-shirt",
```

```
{ "canBePurchased": true, "price": 29.00, "descriptions": [ { "name": "T Shirt", "title": "T Shirt", "language": "en" } ], "identifier": "TSHIRTA", "productSpecifications": { "dimensionUnitOfMeasure": "in", "height": 22, "length": 14, "weight": 0.225, "weightUnitOfMeasure": "lb", "manufacturer": "DEFAULT", "width": 36 }, "shipeable": true, "sortOrder": 10, "type": "GENERAL", "virtual": false, "visible": true } Response: status: OK code: 200 js
```

## get product definition by id

Description:

Product definition can also be immediately attached to category by adding an array of category id in which the product will be created.

```
json "categories": [ { "id": 51 } ],
```

## Method:

GET

## Url:

/api/v2/private/product/26750?lang=en&amp;store=DEFAULT

## Response:

status: OK

## code: 200

js

```
{ "id": 26750, "visible": true, "shipeable": true, "virtual": false, "canBePurchased": true, "dateAvailable": "2022-09-01", "identifier": "TSHIRTA", "sku": null, "productSpecifications": { "height": 22, "weight": 0.23, "length": 14, "width": 36, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "sortOrder": 10, "type": { "id": 300, "code": "GENERAL", "visible": false, "allowAddToCart": false, "description": { "id": 9, "language": "en", "name": "General", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, "categories": [ { "id": 10700, "code": "cotton-shirts", "sortOrder": 0, "visible": true,
```

```
"featured": false, "lineage": "/10700/", "depth": 0, "parent": null, "description": { "id": 371, "language": "en", "name": "Cotton Shirts", "description": null, "friendlyUrl": "cotton-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Standard cotton t-shirts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "manufacturer": { "id": 1, "code": "DEFAULT", "order": 999, "description": { "id": 1, "language": "en", "name": "DEFAULT", "description": "<p>  DEFAULT</p>\n", "friendlyUrl": "", "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, "description": { "id": 25626, "language": "en", "name": "T Shirt", "description": "Light cotton T Shirt", "friendlyUrl": "light-cotton-t-shirt", "keyWords": null,
```

```
"highlights": null, "metaDescription": null, "title": "T Shirt" }, "properties": [], "images": [ { "id": 2400, "imageName": "white-tshirt.jpeg", "imageUrl": "http://localhost:8080/static/products/DEFAULT/TSHI "externalUrl": null, "videoUrl": null, "imageType": 0, "order": 0, "defaultImage": true } ], "inventory": { "id": 26950, "quantity": 100, "region": "*", "regionVariant": null, "owner": null, "dateAvailable": null, "available": true, "productQuantityOrderMin": 0, "productQuantityOrderMax": 0, "creationDate": null, "store": { "id": 1, "code": "DEFAULT", "name": "MCSport", "defaultLanguage": "en", "currency": "SEK", "inBusinessSince": "2021-01-03", "email": "info@mcsport.se", "phone": "08-594 307 70", "template": null, "useCache": true, "currencyFormatNational": false, "retailer": true, "dimension": "CM", "weight": "KG",
```

```
"currentUserLanguage": null, "address": { "stateProvince": "Stockholm", "country": "CA", "address": "V&amp;auml;stbergav&amp;auml;gen 24", "postalCode": "12630", "city": "H&amp;auml;gersten", "active": true }, "logo": null, "parent": null, "supportedLanguages": [ { "code": "en", "id": 1 }, { "code": "sv", "id": 4 } ], "readableAudit": null }, "sku": "TSHIRTA", "prices": [ { "id": 52050, "originalPrice": "SEK26.99", "finalPrice": "SEK25.99", "discounted": false, "description": null } ], "price": "25.99" } }
```

## list product definitions

Description:

Product definition can also be immediately attached to category by adding an array of category id in which the product will be created.

```
json "categories": [ { "id": 51 } ],
```

## Method:

GET

## Url:

/api/v2/products?lang=en&amp;store=DEFAULT&amp;count=25

## Response:

status: OK

## code: 200

js

```
"description": "<ul><li>Kids offroad helmet exclusively for
```

```
{ "totalPages": 8, "number": 25, "recordsTotal": 185, "recordsFiltered": 0, "products": [ { "id": 26000, "productShipeable": false, "available": false, "visible": true, "sortOrder": 0, "dateAvailable": "2022-06-22", "creationDate": "2022-06-22", "price": 7399, "quantity": 40, "sku": "3HS22004720X", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": -1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "3HS22004720X", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25598, "language": "en", "name": "Kids Status Helmet", "friendlyUrl": "kids-status-helmet",
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": "Kids Status Helmet" }, "productPrice": { "id": 51052, "originalPrice": "SEK7,399.00", "finalPrice": "SEK7,399.00", "discounted": false, "description": { "id": 51081, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK7,399.00", "originalPrice": "SEK7,399.00", "discounted": false, "image": null, "images": [ { "id": 2350, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f "videoUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f1-1 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [ {
```

```
"id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": true, "optionValues": [ { "id": 100, "code": "SIZE_HELMET-S-47-48", "name": null, "defaultValue": false, "sortOrder": 20, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "S/47-48", "description": "S/47-48", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, { "id": 101, "code": "SIZE_HELMET-M-49-50", "name": null, "defaultValue": false, "sortOrder": 30, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "M/49-50", "description": "M/49-50",
```

```
"friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, { "id": 102, "code": "SIZE_HELMET-L-51-52", "name": null, "defaultValue": false, "sortOrder": 30, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "L/51-52", "description": "L/51-52", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } } ] } ], "variants": [ { "id": 100, "productShipeable": false, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-22", "creationDate": null, "store": "DEFAULT", "productId": 26000, "sku": "3HS220047202",
```

```
"defaultSelection": false, "variant": { "id": 2, "code": "helmet-size-s-47-48", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 602, "code": "SIZE_HELMET-S-47-48", "name": "S/47-48", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": null, "code": "helmet-size-s-47-48", "images": [], "inventory": [ { "id": 25951, "quantity": 40, "region": "*", "regionVariant": null, "owner": null, "dateAvailable": null, "available": true, "productQuantityOrderMin": -1, "productQuantityOrderMax": 1, "creationDate": null,
```

```
"store": { "id": 1, "code": "DEFAULT", "name": "MCSport", "defaultLanguage": "en", "currency": "SEK", "inBusinessSince": "2021-01-03", "email": "info@mcsport.se", "phone": "08-594 307 70", "template": null, "useCache": true, "currencyFormatNational": false, "retailer": true, "dimension": "CM", "weight": "KG", "currentUserLanguage": null, "address": { "stateProvince": "Stockholm", "country": "CA", "address": "V&amp;auml;stbergav&amp;aum "postalCode": "12630", "city": "H&amp;auml;gersten", "active": true }, "logo": null, "parent": null, "supportedLanguages": [ { "code": "en", "id": 1 }, { "code": "sv", "id": 4 } ], "readableAudit": null }, "sku": "3HS220047202", "prices": [ { "id": 51052, "originalPrice": "SEK7,399.00",
```

```
"finalPrice": "SEK7,399.00", "discounted": false, "description": { "id": 51081, "language": "en", "name": "DEFAULT", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } } ] } ] }, { "id": 101, "productShipeable": false, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-22", "creationDate": null, "store": "DEFAULT", "productId": 26000, "sku": "3HS220047203", "defaultSelection": false, "variant": { "id": 50, "code": "helmet-size-medium", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false, "name": "Size",
```

```
"lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 650, "code": "SIZE_HELMET-M-49-50", "name": "M/49-50", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": null, "code": "helmet-size-medium", "images": [], "inventory": [] }, { "id": 102, "productShipeable": false, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-22", "creationDate": null, "store": "DEFAULT", "productId": 26000, "sku": "3HS220047204", "defaultSelection": false, "variant": { "id": 51, "code": "helmet-size-large", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false,
```

```
"name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 651, "code": "SIZE_HELMET-L-51-52", "name": "L/51-52", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": null, "code": "helmet-size-large", "images": [], "inventory": [] } ], "properties": [], "categories": [ { "id": 10050, "code": "helmets", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10017//10050/", "depth": 1, "parent": { "id": 10017, "code": "equipment" }, "description": { "id": 243, "language": "en", "name": "HELMETS", "description": "", "friendlyUrl": "helmets", "keyWords": "",
```

```
"highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 102, "code": "helmet", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": false, "owner": null }, { "id": 25790, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 2356, "quantity": 1, "sku": "21008915044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm",
```

```
"weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "21008915044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25518, "language": "en", "name": "Licence plate holder support ", "description": "<ul><li>Visual highlight</li><li>Perfectly "friendlyUrl": "licence-plate-holder-support-", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Licence plate holder support " }, "productPrice": { "id": 50830, "originalPrice": "SEK2,356.00", "finalPrice": "SEK2,356.00", "discounted": false, "description": { "id": 50919, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK2,356.00", "originalPrice": "SEK2,356.00", "discounted": false, "image": null, "images": [ { "id": 2263,
```

```
"imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/7/7fbcf2b "videoUrl": "https://cms.jinomotos.com/pim/7/7fbcf2b0-9 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT", "children": [] }
```

],

```
"type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25824, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1645, "quantity": 1, "sku": "23607940200", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "23607940200", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25586,
```

```
"language": "en", "name": "Seat ", "description": "<ul><li>Resistant coating</li><li>Enables s "friendlyUrl": "seat-", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Seat " }, "productPrice": { "id": 50898, "originalPrice": "SEK1,645.00", "finalPrice": "SEK1,645.00", "discounted": false, "description": { "id": 51054, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,645.00", "originalPrice": "SEK1,645.00", "discounted": false, "image": null, "images": [ { "id": 2302, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/9/9076af3 "videoUrl": "https://cms.jinomotos.com/pim/9/9076af33-7 "imageType": 1, "order": 0, "defaultImage": true }
```

],

```
"manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10353, "code": "seats", "sortOrder": 310, "visible": true, "featured": false, "lineage": "/10150//10003//10353/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 299, "language": "en", "name": "Seat & covers", "description": "", "friendlyUrl": "seat-covers", "keyWords": "", "highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null
```

```
}, { "id": 25812, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1645, "quantity": 1, "sku": "25007940000", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25007940000", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25566, "language": "en", "name": "Seat", "description": "<ul><li>Resistant coating</li><li>Higher co "friendlyUrl": "seat", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Seat" },
```

```
"productPrice": { "id": 50874, "originalPrice": "SEK1,645.00", "finalPrice": "SEK1,645.00", "discounted": false, "description": { "id": 51007, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,645.00", "originalPrice": "SEK1,645.00", "discounted": false, "image": null, "images": [ { "id": 2292, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/b/b8a244e "videoUrl": "https://cms.jinomotos.com/pim/b/b8a244e0-1 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10353, "code": "seats",
```

```
"sortOrder": 310, "visible": true, "featured": false, "lineage": "/10150//10003//10353/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 299, "language": "en", "name": "Seat & covers", "description": "", "friendlyUrl": "seat-covers", "keyWords": "", "highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25813, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28",
```

```
"price": 1645, "quantity": 1, "sku": "25007940100", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25007940100", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25562, "language": "en", "name": "Seat", "description": "<ul><li>Resistant coating</li><li>Enables s "friendlyUrl": "seat", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Seat" }, "productPrice": { "id": 50875, "originalPrice": "SEK1,645.00", "finalPrice": "SEK1,645.00", "discounted": false, "description": { "id": 51009, "language": "en", "name": null,
```

```
"description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,645.00", "originalPrice": "SEK1,645.00", "discounted": false, "image": null, "images": [ { "id": 2290, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/6/6c1e621 "videoUrl": "https://cms.jinomotos.com/pim/6/6c1e6217-b "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10353, "code": "seats", "sortOrder": 310, "visible": true, "featured": false, "lineage": "/10150//10003//10353/", "depth": 2, "parent": { "id": 10003, "code": "chassis" },
```

```
"description": { "id": 299, "language": "en", "name": "Seat & covers", "description": "", "friendlyUrl": "seat-covers", "keyWords": "", "highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25787, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 651, "quantity": 1, "sku": "2500805400028", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": {
```

```
"height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "2500805400028", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25512, "language": "en", "name": "Spoiler set", "description": "null", "friendlyUrl": "spoiler-set", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Spoiler set" }, "productPrice": { "id": 50824, "originalPrice": "SEK651.00", "finalPrice": "SEK651.00", "discounted": false, "description": { "id": 50906, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null }
```

},

```
"finalPrice": "SEK651.00", "originalPrice": "SEK651.00", "discounted": false, "image": null, "images": [ { "id": 2260, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/d/da12175 "videoUrl": "https://cms.jinomotos.com/pim/d/da121755-c "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null,
```

```
"title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25782, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 651, "quantity": 1, "sku": "25008054000HA", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" },
```

```
"rating": 0, "ratingCount": 0, "refSku": "25008054000HA", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25502, "language": "en", "name": "Spoiler set", "description": "null", "friendlyUrl": "spoiler-set", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Spoiler set" }, "productPrice": { "id": 50814, "originalPrice": "SEK651.00", "finalPrice": "SEK651.00", "discounted": false, "description": { "id": 50887, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK651.00", "originalPrice": "SEK651.00", "discounted": false, "image": null, "images": [ { "id": 2254, "imageName": null,
```

```
"imageUrl": "http://localhost:8080/static/products/DEFA
```

```
"externalUrl": "https://cms.jinomotos.com/pim/9/90c2ddb "videoUrl": "https://cms.jinomotos.com/pim/9/90c2ddb5-c "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150,
```

```
"code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25701, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 295, "quantity": 1, "sku": "25013960000", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25013960000", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25334, "language": "en", "name": "Footbrake cylinder extender",
```

```
"description": "<ul><li>Increases the volume of the rear ma "friendlyUrl": "footbrake-cylinder-extender", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Footbrake cylinder extender" }, "productPrice": { "id": 50649, "originalPrice": "SEK295.00", "finalPrice": "SEK295.00", "discounted": false, "description": { "id": 50554, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK295.00", "originalPrice": "SEK295.00", "discounted": false, "image": null, "images": [ { "id": 2156, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/3/3b9b768 "videoUrl": "https://cms.jinomotos.com/pim/3/3b9b7685-7 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [],
```

```
"options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null },
```

{

```
"id": 25822, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1645, "quantity": 1, "sku": "25107940100", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25107940100", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25581, "language": "en", "name": "Seat", "description": "<ul><li>Resistant coating</li><li>Higher co "friendlyUrl": "seat", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Seat" }, "productPrice": { "id": 50894,
```

```
"originalPrice": "SEK1,645.00", "finalPrice": "SEK1,645.00", "discounted": false, "description": { "id": 51046, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,645.00", "originalPrice": "SEK1,645.00", "discounted": false, "image": null, "images": [ { "id": 2300, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/d/dfa2657 "videoUrl": "https://cms.jinomotos.com/pim/d/dfa26577-9 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10353, "code": "seats", "sortOrder": 310, "visible": true,
```

```
"featured": false, "lineage": "/10150//10003//10353/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 299, "language": "en", "name": "Seat & covers", "description": "", "friendlyUrl": "seat-covers", "keyWords": "", "highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25816, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1645, "quantity": 1,
```

```
"sku": "25107940200", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25107940200", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25569, "language": "en", "name": "Seat", "description": "<ul><li>Resistant coating</li><li>Enables s "friendlyUrl": "seat", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Seat" }, "productPrice": { "id": 50878, "originalPrice": "SEK1,645.00", "finalPrice": "SEK1,645.00", "discounted": false, "description": { "id": 51008, "language": "en", "name": null, "description": null, "friendlyUrl": null,
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,645.00", "originalPrice": "SEK1,645.00", "discounted": false, "image": null, "images": [ { "id": 2293, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/0/0805282 "videoUrl": "https://cms.jinomotos.com/pim/0/0805282e-0 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10353, "code": "seats", "sortOrder": 310, "visible": true, "featured": false, "lineage": "/10150//10003//10353/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 299,
```

```
"language": "en", "name": "Seat & covers", "description": "", "friendlyUrl": "seat-covers", "keyWords": "", "highlights": "", "metaDescription": "", "title": "" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25721, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 344, "quantity": 2, "sku": "25530902144", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null,
```

```
"length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25530902144", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25384, "language": "en", "name": "Factory Racing ignition cover plug", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "factory-racing-ignition-cover-plug", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Factory Racing ignition cover plug" }, "productPrice": { "id": 50690, "originalPrice": "SEK344.00", "finalPrice": "SEK344.00", "discounted": false, "description": { "id": 50643, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK344.00", "originalPrice": "SEK344.00",
```

```
"discounted": false, "image": null, "images": [ { "id": 2189, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/4/48aa44c "videoUrl": "https://cms.jinomotos.com/pim/4/48aa44ca-d "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" },
```

```
"productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25708, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 533, "quantity": 1, "sku": "25534931044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0,
```

```
"refSku": "25534931044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25353, "language": "en", "name": "Gear shift lever step plate", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "gear-shift-lever-step-plate", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Gear shift lever step plate" }, "productPrice": { "id": 50666, "originalPrice": "SEK533.00", "finalPrice": "SEK533.00", "discounted": false, "description": { "id": 50590, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK533.00", "originalPrice": "SEK533.00", "discounted": false, "image": null, "images": [ { "id": 2166, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/0/05501c2 "videoUrl": "https://cms.jinomotos.com/pim/0/05501c23-9
```

```
"imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true,
```

```
"allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25732, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 462, "quantity": 1, "sku": "25538941044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "25538941044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25402, "language": "en", "name": "Factory Racing oil filter cover", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "factory-racing-oil-filter-cover",
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": "Factory Racing oil filter cover" }, "productPrice": { "id": 50714, "originalPrice": "SEK462.00", "finalPrice": "SEK462.00", "discounted": false, "description": { "id": 50686, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK462.00", "originalPrice": "SEK462.00", "discounted": false, "image": null, "images": [ { "id": 2199, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/5/5b645f0 "videoUrl": "https://cms.jinomotos.com/pim/5/5b645f06-8 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [],
```

```
"properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25725, "productShipeable": true,
```

```
"available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 236, "quantity": 1, "sku": "26130920044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "26130920044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25387, "language": "en", "name": "Oil plug", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "oil-plug", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Oil plug" }, "productPrice": { "id": 50700, "originalPrice": "SEK236.00", "finalPrice": "SEK236.00",
```

```
"discounted": false, "description": { "id": 50658, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK236.00", "originalPrice": "SEK236.00", "discounted": false, "image": null, "images": [ { "id": 2191, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/3/3ea4bfe "videoUrl": "https://cms.jinomotos.com/pim/3/3ea4bfe2-4 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/",
```

```
"depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25663, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 236, "quantity": 8, "sku": "26602908000", "preOrder": false,
```

```
"productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "26602908000", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25264, "language": "en", "name": "Handlebar muffs", "description": "<ul><li>Never again cold fingers!</li><li>M "friendlyUrl": "handlebar-muffs", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebar muffs" }, "productPrice": { "id": 50576, "originalPrice": "SEK236.00", "finalPrice": "SEK236.00", "discounted": false, "description": { "id": 50410, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null,
```

```
"metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK236.00", "originalPrice": "SEK236.00", "discounted": false, "image": null, "images": [ { "id": 2114, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/9/9df2429 "videoUrl": "https://cms.jinomotos.com/pim/9/9df24299-f "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346, "language": "en", "name": "Hand Guards",
```

```
"description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25743, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 461, "quantity": 1, "sku": "26606001000DB", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null,
```

```
"model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "26606001000DB", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25424, "language": "en", "name": "Lower section of the air filter ", "description": "null", "friendlyUrl": "lower-section-of-the-air-filter-", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Lower section of the air filter " }, "productPrice": { "id": 50736, "originalPrice": "SEK461.00", "finalPrice": "SEK461.00", "discounted": false, "description": { "id": 50731, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK461.00", "originalPrice": "SEK461.00", "discounted": false, "image": null,
```

```
"images": [ { "id": 2211, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/6/6619d3a "videoUrl": "https://cms.jinomotos.com/pim/6/6619d3a2-4 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT",
```

```
"children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25784, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 344, "quantity": 1, "sku": "26608001000DB", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "26608001000DB", "rentalDuration": 0,
```

```
"rentalPeriod": 0, "description": { "id": 25506, "language": "en", "name": "Headlight shroud", "description": "null", "friendlyUrl": "headlight-shroud", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Headlight shroud" }, "productPrice": { "id": 50818, "originalPrice": "SEK344.00", "finalPrice": "SEK344.00", "discounted": false, "description": { "id": 50894, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK344.00", "originalPrice": "SEK344.00", "discounted": false, "image": null, "images": [ { "id": 2256, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/f/f256a3b "videoUrl": "https://cms.jinomotos.com/pim/f/f256a3b2-6 "imageType": 1, "order": 0,
```

```
"defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null
```

```
}, "canBePurchased": true, "owner": null }, { "id": 25793, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1408, "quantity": 5, "sku": "27008915044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "27008915044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25523, "language": "en", "name": "Licence plate holder support ", "description": "<ul><li>Visual highlight</li><li>Perfectly "friendlyUrl": "licence-plate-holder-support-", "keyWords": null, "highlights": null,
```

```
"metaDescription": null, "title": "Licence plate holder support " }, "productPrice": { "id": 50836, "originalPrice": "SEK1,408.00", "finalPrice": "SEK1,408.00", "discounted": false, "description": { "id": 50931, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,408.00", "originalPrice": "SEK1,408.00", "discounted": false, "image": null, "images": [ { "id": 2266, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/4/444a28a "videoUrl": "https://cms.jinomotos.com/pim/4/444a28a1-0 "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2267, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/7/72bf208 "videoUrl": "https://cms.jinomotos.com/pim/7/72bf2086-0 "imageType": 1,
```

```
"order": 0, "defaultImage": false } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10605, "code": "plastic-parts", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10605/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 356, "language": "en", "name": "Separate Plastic Parts", "description": null, "friendlyUrl": "separate-plastic-parts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Separate Plastic Parts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true,
```

```
"description": null }, "canBePurchased": true, "owner": null }, { "id": 25710, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 296, "quantity": 2, "sku": "27013903000", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "27013903000", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25358, "language": "en", "name": "Brake fluid reservoir cover", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "brake-fluid-reservoir-cover", "keyWords": null,
```

```
"highlights": null, "metaDescription": null, "title": "Brake fluid reservoir cover" }, "productPrice": { "id": 50670, "originalPrice": "SEK296.00", "finalPrice": "SEK296.00", "discounted": false, "description": { "id": 50598, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK296.00", "originalPrice": "SEK296.00", "discounted": false, "image": null, "images": [ { "id": 2168, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/3/3c5123f "videoUrl": "https://cms.jinomotos.com/pim/3/3c5123f5-2 "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2169, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/8/8629c65 "videoUrl": "https://cms.jinomotos.com/pim/8/8629c656-e
```

```
"imageType": 1, "order": 0, "defaultImage": false } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true,
```

```
"allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25667, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 699, "quantity": 1, "sku": "6030217910030", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "6030217910030", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25271, "language": "en", "name": "Handguard kit",
```

```
"description": "<ul><li>Excellent weather protection</li><l "friendlyUrl": "handguard-kit",
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": "Handguard kit" }, "productPrice": { "id": 50584, "originalPrice": "SEK699.00", "finalPrice": "SEK699.00", "discounted": false, "description": { "id": 50426, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK699.00", "originalPrice": "SEK699.00", "discounted": false, "image": null, "images": [ { "id": 2119, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/9/9e1dfb6 "videoUrl": "https://cms.jinomotos.com/pim/9/9e1dfb6a-a "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [],
```

```
"properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346, "language": "en", "name": "Hand Guards", "description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25656, "productShipeable": true,
```

```
"available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1054, "quantity": 6, "sku": "76502979144", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "76502979144", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25247, "language": "en", "name": "Wrap-around handguard kit", "description": "<ul><li>Including all required assembly mat "friendlyUrl": "wrap-around-handguard-kit", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Wrap-around handguard kit" }, "productPrice": { "id": 50559, "originalPrice": "SEK1,054.00", "finalPrice": "SEK1,054.00",
```

```
"discounted": false, "description": { "id": 50378, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,054.00", "originalPrice": "SEK1,054.00", "discounted": false, "image": null, "images": [ { "id": 2106, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/4/4e91e48 "videoUrl": "https://cms.jinomotos.com/pim/4/4e91e489-3 "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2502, "imageName": "blob", "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": null, "videoUrl": null, "imageType": 0, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [],
```

```
"variants": [], "properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346, "language": "en", "name": "Hand Guards", "description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25687,
```

```
"productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 651, "quantity": 1, "sku": "77701939010", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "77701939010", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25313, "language": "en", "name": "Handlebar riser", "description": "<p></p>These CNC-machined, high-strength al "friendlyUrl": "handlebar-riser", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebar riser" }, "productPrice": { "id": 50624, "originalPrice": "SEK651.00",
```

```
"finalPrice": "SEK651.00", "discounted": false, "description": { "id": 50506, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK651.00", "originalPrice": "SEK651.00", "discounted": false, "image": null, "images": [ { "id": 2141, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/a/a5114a0 "videoUrl": "https://cms.jinomotos.com/pim/a/a5114a0d-c "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10601, "code": "handlebar-support", "sortOrder": 0, "visible": true, "featured": false,
```

```
"lineage": "/10150//10003/10601/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 349, "language": "en", "name": "Handlebar Support", "description": null, "friendlyUrl": "handlebar-support", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebar Support" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25714, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 580, "quantity": 2, "sku": "77734931144",
```

```
"preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "77734931144", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25366, "language": "en", "name": "Gear shift lever step plate", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "gear-shift-lever-step-plate", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Gear shift lever step plate" }, "productPrice": { "id": 50678, "originalPrice": "SEK580.00", "finalPrice": "SEK580.00", "discounted": false, "description": { "id": 50614, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null,
```

```
"highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK580.00", "originalPrice": "SEK580.00", "discounted": false, "image": null, "images": [ { "id": 2177, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/f/fa7147f "videoUrl": "https://cms.jinomotos.com/pim/f/fa7147f7-2 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en",
```

```
"name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25655, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1172, "quantity": 3, "sku": "79702984044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null,
```

```
"width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "79702984044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25244, "language": "en", "name": "Wrap-around handguard kit", "description": "<ul><li>Including all required assembly mat "friendlyUrl": "wrap-around-handguard-kit", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Wrap-around handguard kit" }, "productPrice": { "id": 50558, "originalPrice": "SEK1,172.00", "finalPrice": "SEK1,172.00", "discounted": false, "description": { "id": 50372, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,172.00", "originalPrice": "SEK1,172.00", "discounted": false,
```

```
"image": null, "images": [ { "id": 2103, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/7/7b75de5 "videoUrl": "https://cms.jinomotos.com/pim/7/7b75de5b-c "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2104, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/2/285a696 "videoUrl": "https://cms.jinomotos.com/pim/2/285a6966-7 "imageType": 1, "order": 0, "defaultImage": false } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346,
```

```
"language": "en", "name": "Hand Guards", "description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }, { "id": 25715, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 296, "quantity": 1, "sku": "81302933044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null,
```

```
"length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "81302933044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25368, "language": "en", "name": "Clutch reservoir cover", "description": "<ul><li>CNC milled from high-strength alumi "friendlyUrl": "clutch-reservoir-cover", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Clutch reservoir cover" }, "productPrice": { "id": 50680, "originalPrice": "SEK296.00", "finalPrice": "SEK296.00", "discounted": false, "description": { "id": 50618, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK296.00", "originalPrice": "SEK296.00",
```

```
"discounted": false, "image": null, "images": [ { "id": 2178, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/7/7f7484e "videoUrl": "https://cms.jinomotos.com/pim/7/7f7484e4-8 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10603, "code": "bling", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10003/10603/", "depth": 2, "parent": { "id": 10003, "code": "chassis" }, "description": { "id": 352, "language": "en", "name": "Bling Bling", "description": null, "friendlyUrl": "bling-bling", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Handlebars" },
```

```
"productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null } ] }
```

## delete product definition

## Description:

Product definition can also be immediately attached to category by adding an array of category id in which the product will be created.

```
json "categories": [ { "id": 51 } ],
```

## Method:

DELETE

## Url:

/api/v2/private/product/26700

## Response:

status: OK

code: 200

## 9. api/products/product-images.html

## URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-images.html

## Product images

Product images management.

## Add product images

## Description:

## Method:

POST

## Url:

/api/v1/private/product/1/image

## Response: status: Created code: 201 js

## List product images

## Description:

List product images

## Method:

GET Url:

/api/v1/product/1/images

```
Response: status: OK code: 200 js [ { "id": 2, "imageName": "white-tshirt.jpeg", "imageUrl": "http://localhost:80/products/DEFAULT/TSHIRTA/SMALL/whi "externalUrl": null, "videoUrl": null, "imageType": 0, "order": 0, "defaultImage": true } ]
```

## Update product image sort order

## Description:

Updates an image sort order

## Method:

PATCH

## Url:

/api/v1/private/product/1/image/1?order=1

Response: status: OK code: 200

## Delete product image

## Description:

Updates an image sort order

## Method:

DELETE

## Url:

/api/v1/private/product/1/image/1

Response: status: OK code: 200 js

## 10. api/products/product-instances-group.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-instances-group.html

## Product Instances Group

ProductInstances can be part of a ProductInstanceGroup allowing to share common resources. Example: all white T Shirts product instances are part of o common group with an image of a white T

Shirt.

## Create Product Instance Group

## Description:

Create a product instance group using all product instances id in an array

## Method:

POST

## Url:

/api/v2/private/product/productInstanceGroup

```
Body: js { "productInstances": [1003,1002] }
```

```
Response: status: Created code: 201 js { "id": 50 }
```

## Update Product Instance Group

## Description:

Edit product instance group for adding and removing product instance id

## Method:

PUT

## Url:

/api/v2/private/product/productInstanceGroup/50

## Body:

```
{ "productInstances": [1003]
```

```
js } Response: status: OK code: 200 js
```

## List Product Instances Group

## Description:

List product instances group for a given store and a specific product definition.

## Method:

GET

## Url:

/api/v2/private/product/26650/productInstanceGroup?store=DEFAULT

## Response:

status: OK

code: 200

js

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "items": [ { "id": 50, "images": [], "productInstances": [ { "id": 1002, "productShipeable": true, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-23", "creationDate": null, "store": "DEFAULT", "productId": 26650, "sku": "A256TMW", "defaultSelection": false, "variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false,
```

```
"sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": { "id": 100, "code": "size-medium", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 700, "code": "medium", "name": "Medium", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-medium", "images": [], "inventory": [] }, { "id": 1003, "productShipeable": true, "available": false, "visible": true, "sortOrder": 10,
```

```
"dateAvailable": "2022-02-23", "creationDate": null, "store": "DEFAULT", "productId": 26650, "sku": "A256TLW", "defaultSelection": false, "variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": { "id": 150, "code": "size-large", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select",
```

```
"readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 750, "code": "large", "name": "Large", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-large", "images": [], "inventory": [] } ] } ] }
```

## Get Product Instance Group

## Description:

Get product instance group details

## Method:

GET

## Url:

/api/v2/private/product/productInstanceGroup/50

## Response:

status: OK

js

```
{ "id": 50, "images": [], "productInstances": [ { "id": 1002, "productShipeable": true, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-23", "creationDate": null, "store": "DEFAULT", "productId": 26650, "sku": "A256TMW", "defaultSelection": false, "variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } },
```

```
"variantValue": { "id": 100, "code": "size-medium", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 700, "code": "medium", "name": "Medium", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-medium", "images": [], "inventory": [] }, { "id": 1003, "productShipeable": true, "available": false, "visible": true, "sortOrder": 10, "dateAvailable": "2022-02-23", "creationDate": null, "store": "DEFAULT", "productId": 26650, "sku": "A256TLW", "defaultSelection": false,
```

```
"variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": { "id": 150, "code": "size-large", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] },
```

```
"optionValue": { "id": 750, "code": "large", "name": "Large", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-large", "images": [], "inventory": [] } ] }
```

## Add Product Instance Group images

## Description:

Add an image to the product instance group

## Method:

POST

## Url:

/api/v2/private/product/productInstanceGroup/51/image

## Header

```
"key": "Content-Type", "value": "multipart/form-data"
```

## Payload

```
"formdata": [ { "key": "file", "type": "file", "src": "/Users/carlsamson/Documents/csti/shopizer/images/misc/whi }, { "order": "0", }
```

Response: status: Created code: 201 js

## Delete Product Instance Group

## Description:

Delete a product instance group

## Method:

DELETE

## Url:

/api/v2/private/product/productInstanceGroup/1

## Response:

status: OK

code: 200

js

## Delete Product Instance Group Image

## Description:

Remove an image to a product instance group Method: DELETE Url: /api/v2/private/product/productInstanceGroup/51/image/1 Response: status: OK code: 200

## js

## 11. api/products/product-instances.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-instances.html

## Product Variants

A Product Variant is a product definition with a SKU and Variants. A Product definition will become a Product variant when variants are attached to it. An example for this is a simple Product variant (TShirt) will possibly have product variant such as TShirt Medium Black or TShirt Small White.

The product variant will be decorated with variants and will have a specific SKU attached to it.

## Create Product variant

## Description:

```
Creates a product variant Method: POST Url: /api/v2/private/product/26100/variant Body: js { "visible": true, "dateAvailable": "2022-07-01", "sortOrder": 1, "sku" : "A256TSW", "variant": 4, "variantValue": 100 } Response: status: Created code: 201 js "id": 150
```

```
{ }
```

## Update Product variant

Description:

```
Method: PUT Url: /api/v2/private/product/26100/variant/150 Body: js { "visible": true, "dateAvailable": "2022-07-01", "sortOrder": 2, "sku" : "A256TSW", "variant": 4, "variantValue": 100 } Response: status: OK code: 200 js
```

## Check if Product variant exists

Description:

Method:

GET

Url:

js { "exists": true

```
Response: status: OK code: 200 }
```

## List product variants

## Description:

## Method:

GET

## Url:

/api/v2/private/product/26100/variants

## Response:

## status: OK

## code: 200

js

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "items": [ { "id": 150, "productShipeable": true, "available": false, "visible": true, "sortOrder": 2, "dateAvailable": "2022-07-01", "creationDate": null, "store": "DEFAULT", "productId": 26100, "sku": "A256TSW", "defaultSelection": false, "variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null
```

```
} }, "variantValue": { "id": 100, "code": "size-medium", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 700, "code": "medium", "name": "Medium", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-medium", "images": [], "inventory": [] } ] }
```

## Get Product variant

Description:

Method:

GET

## Url:

/api/v2/private/product/26100/variant/150

## Response:

## status: OK

## code: 200

js

```
{ "id": 150, "productShipeable": true, "available": false, "visible": true, "sortOrder": 2, "dateAvailable": "2022-07-01", "creationDate": null, "store": "DEFAULT", "productId": 26100, "sku": "A256TSW", "defaultSelection": false, "variant": { "id": 4, "code": "colour-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 100, "code": "COLOUR", "type": "select", "readOnly": false, "name": "Colour", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 600, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": { "id": 100, "code": "size-medium", "date": null,
```

```
"sortOrder": 0, "defaultValue": false, "option": { "id": 150, "code": "size", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 700, "code": "medium", "name": "Medium", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "code": "colour-white:size-medium", "images": [], "inventory": [] }
```

## Delete Product variant

Description:

Method:

DELETE

Url:

/api/v2/private/product/26100/variant/150

Response:

status: OK

code: 200 js

## 12. api/products/product-listing.html

## URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-listing.html

## Product Listing

API for listing products based on different filters. All requests accept paging parameters 'page=' and expected page size 'count='. All query also accept the store 'store=' and language 'language=`'. Response will display the following fields: "totalPages": 10, "number": 20, "recordsTotal": 185, totalPages is the number of expected pages based on the number of records. number is the count sent in the request. recordsTotal is the total number of item in storage. Filters can be combined in query for instance: /api/v2/products?manufacturerId=100&amp;optionValueCodes=white,large

## Page products (list)

## Description:

List all products single or multiple filters can be applied.

## Method:

GET Url:

## Response: status: OK

## code: 200 js

```
"description": "<ul><li>Kids offroad helmet exclusively for
```

```
{ "totalPages": 10, "number": 20, "recordsTotal": 185, "recordsFiltered": 0, "products": [ { "id": 26000, "productShipeable": false, "available": false, "visible": true, "sortOrder": 0, "dateAvailable": "2022-06-22", "creationDate": "2022-06-22", "price": 7399, "quantity": 40, "sku": "3HS22004720X", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": -1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "3HS22004720X", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25598, "language": "en", "name": "Kids Status Helmet", "friendlyUrl": "kids-status-helmet",
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": "Kids Status Helmet" }, "productPrice": { "id": 51052, "originalPrice": "SEK7,399.00", "finalPrice": "SEK7,399.00", "discounted": false, "description": { "id": 51081, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK7,399.00", "originalPrice": "SEK7,399.00", "discounted": false, "image": null, "images": [ { "id": 2350, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f "videoUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f1-1 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], ...,
```

## List items in specific category

## Description:

List all products filter by category (1 to multiple)

## Method:

GET

## Url:

/api/v12products

## Response: status: OK code: 200 js

```
{ "totalPages": 1, "number": 2, "recordsTotal": 2, "recordsFiltered": 0, "products": [ { "id": 26850, "productShipeable": true, "available": true, "visible": true, "sortOrder": 5, "dateAvailable": "2022-09-02", "creationDate": "2022-09-04", "price": 34, "quantity": 1000, "sku": "PRBSK01", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": 12, "weight": 250, "length": 8, "width": 14, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": null, "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25628, "language": "en", "name": "Nike Sport Shirt", "description": "Nike sport shirt", "friendlyUrl": "nike-sport-shirt",
```

```
"keyWords": "", "highlights": "", "metaDescription": "", "title": "Nike Sport Shirt" }, "productPrice": { "id": 51900, "originalPrice": "SEK34.00", "finalPrice": "SEK34.00", "discounted": false, "description": { "id": 51111, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK34.00", "originalPrice": "SEK34.00", "discounted": false, "image": null, "images": [], "manufacturer": { "id": 100, "code": "nike", "order": 0, "description": { "id": 32, "language": "en", "name": "Nike", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null }
```

```
}, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10800, "code": "sport-shirts", "sortOrder": 10, "visible": true, "featured": false, "lineage": "/10800/", "depth": 0, "parent": null, "description": { "id": 373, "language": "en", "name": "Training Shirts", "description": null, "friendlyUrl": "training-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Training Shirts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null },
```

...

] }

## List items having a name like

## Description:

List all products filtered by name

## Method:

GET

## Url:

localhost:8080/api/v1/products

## Response: status: OK code: 200 js

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "products": [ { "id": 26850, "productShipeable": true, "available": true, "visible": true, "sortOrder": 5, "dateAvailable": "2022-09-02", "creationDate": "2022-09-04", "price": 34, "quantity": 1000, "sku": "PRBSK01", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": 12, "weight": 250, "length": 8, "width": 14, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, ... ] }
```

## List items filtered by manufacturer

## Description:

List all products filter by a specific category id.

## Method:

GET

## Url:

localhost:8080/api/v1/products

## Response: status: OK code: 200 js

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "products": [ { "id": 26900, "productShipeable": true, "available": true, "visible": true, "sortOrder": 7, "dateAvailable": "2022-09-06", "creationDate": "2022-09-06", "price": 32, "quantity": 0, "sku": "ATRS01", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "canBePurchased": false, "owner": null, ... } ] }
```

## List items filtered by option value id(s)

## Description:

List all products filter by product option value id (1 .. n)

## Method:

GET

Url:

```
Response: status: OK code: 200 js { "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "products": [ { "id": 26750, "productShipeable": true, "available": true, ... }, "canBePurchased": false, "owner": null } ] }
```

## List items filtered by option value code(s)

## Description:

List products filter by option value code.

## Method:

GET

## Url:

/api/v2/products?optionValueCodes=coton

## Response:

status: OK

js

```
"description": "<ul><li>Kids offroad helmet exclusively for
```

```
{ "totalPages": 2, "number": 100, "recordsTotal": 185, "recordsFiltered": 0, "products": [ { "id": 26000, "productShipeable": false, "available": false, "visible": true, "sortOrder": 0, "dateAvailable": "2022-06-22", "creationDate": "2022-06-22", "price": 7399, "quantity": 40, "sku": "3HS22004720X", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": -1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "3HS22004720X", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25598, "language": "en", "name": "Kids Status Helmet", "friendlyUrl": "kids-status-helmet",
```

```
"keyWords": null, "highlights": null, "metaDescription": null, "title": "Kids Status Helmet" }, "productPrice": { "id": 51052, "originalPrice": "SEK7,399.00", "finalPrice": "SEK7,399.00", "discounted": false, "description": { "id": 51081, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK7,399.00", "originalPrice": "SEK7,399.00", "discounted": false, "image": null, "images": [ { "id": 2350, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFA "externalUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f "videoUrl": "https://cms.jinomotos.com/pim/e/e9c5e3f1-1 "imageType": 1, "order": 0, "defaultImage": true } ], "manufacturer": null, "attributes": [], "options": [ {
```

```
"id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": true, "optionValues": [ { "id": 100, "code": "SIZE_HELMET-S-47-48", "name": null, "defaultValue": false, "sortOrder": 20, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "S/47-48", "description": "S/47-48", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, { "id": 101, "code": "SIZE_HELMET-M-49-50", "name": null, "defaultValue": false, "sortOrder": 30, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "M/49-50", "description": "M/49-50",
```

```
"friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, { "id": 102, "code": "SIZE_HELMET-L-51-52", "name": null, "defaultValue": false, "sortOrder": 30, "image": null, "order": 0, "price": null, "description": { "id": 0, "language": "en", "name": "L/51-52", "description": "L/51-52", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } } ] } ], "variants": [ { "id": 100, "productShipeable": false, "available": false, "visible": true, "sortOrder": 1, "dateAvailable": "2022-02-22", "creationDate": null, "store": "DEFAULT", "productId": 26000, "sku": "3HS220047202",
```

```
"defaultSelection": false, "variant": { "id": 2, "code": "helmet-size-s-47-48", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 102, "code": "SIZE_HELMET", "type": "select", "readOnly": false, "name": "Size", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 602, "code": "SIZE_HELMET-S-47-48", "name": "S/47-48", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }, "variantValue": null, "code": "helmet-size-s-47-48", "images": [], "inventory": [ { "id": 25951, "quantity": 40, "region": "*", "regionVariant": null, "owner": null, "dateAvailable": null, "available": true, "productQuantityOrderMin": -1, "productQuantityOrderMax": 1, "creationDate": null,
```

```
"store": { "id": 1, "code": "DEFAULT", "name": "MCSport", "defaultLanguage": "en", "currency": "SEK", "inBusinessSince": "2021-01-03", "email": "info@mcsport.se", "phone": "08-594 307 70", "template": null, "useCache": true, "currencyFormatNational": false, "retailer": true, "dimension": "CM", "weight": "KG", "currentUserLanguage": null, "address": { "stateProvince": "Stockholm", "country": "CA", "address": "V&amp;auml;stbergav&amp;aum "postalCode": "12630", "city": "H&amp;auml;gersten", "active": true }, "logo": null, "parent": null, "supportedLanguages": [ { "code": "en", "id": 1 }, { "code": "sv", "id": 4 } ], "readableAudit": null }, "sku": "3HS220047202", "prices": [ { "id": 51052, "originalPrice": "SEK7,399.00",
```

```
"finalPrice": "SEK7,399.00", "discounted": false, "description": { "id": 51081, "language": "en", "name": "DEFAULT", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } } ] } ] }, ... ] }
```

## 13. api/products/product-options.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-options.html

## Product Options

Product Options can be used for creating product variants that customer can select.

## Create option

## Description:

API for the creation of a product option

```
Method: POST Url: /api/v1/private/product/option Body: js { "code": "color", "descriptions": [ { "name": "Color", "language": "en" }, { "name": "Couleur", "language": "fr" } ], "order": 1, "type": "radio" } Response: status: Created code: 201 js
```

```
{ "id": 1, "code": "color", "type": "radio", "readOnly": false, "order": 0, "description": { "id": 1, "language": "en", "name": "Color", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value

## Description:

API for the creation of a product option

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

js

```
{ "code": "white", "descriptions": [ { }, { } ], "order": 0
```

```
"name": "White", "language": "en" "name": "Blanc", "language": "fr" } Response: status: Created code: 201 js
```

```
{ "id": 1, "code": "white", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 1, "language": "en", "name": "White", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Get product options

## Description:

Get product options

## Method:

GET

## Url:

/api/v1/private/product/options

## Response:

status: OK

code: 200

```
{ "totalPages": 1, "number": 0, "recordsTotal": 2, "recordsFiltered": 0, "options": [ { "id": 1, "code": "color", "type": "radio", "readOnly": false, "order": 0, "description": null, "descriptions": [ { "id": 2, "language": "fr", "name": "Couleur", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null }, { "id": 1, "language": "en", "name": "Color", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } ] }, { "id": 3, "code": "size", "type": "select", "readOnly": false,
```

```
"order": 0, "description": null, "descriptions": [ { "id": 5, "language": "en", "name": "Size", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null }, { "id": 6, "language": "fr", "name": "Grandeur", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } ] } ] }
```

## Udate product option

## Description:

API for the update of a product option The same applies for editing a product option value

PUT /api/v1/product/option/value

## Method:

POST

## Url:

```
/api/v1/private/product/ Body: js "name": "Grandeur",
```

```
{ "code": "size", "descriptions": [ { "name": "Size", "language": "en" }, { "language": "fr" } ], "order": 4, "type": "select" }
```

## Delete product option

## Description:

API for deleting a product option

The same applies for deleting a product option value

DELETE /api/v1/product/option/value

** If an option is attached to a product variants or attribute, the attachment should be removed first

## Method:

DELETE

## Url:

/api/v1/private/product/

## 14. api/products/product-properties.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-properties.html

## Product Properties

Properties (attributes) adds extension to the product core attributes by allowing the creation of custom properties (attributes).

## Create option

## Description:

API for the creation of a product option. A product option is necessary for creating a product attribute. readonly field for a product option dedicated to become a product attribute must have readonly field set to true

Examples of product options for attribute:

- Maximum Heat
- Minimum Heat
- Weight Capacity

-...

## Method:

POST

## Url:

/api/v1/private/product/option

## Body:

js

```
{ "code": "fabric", "readonly": true, "descriptions": [ { "name": "Fabric", "language": "en" }, { "language": "fr" } ]
```

```
"name": "Matériel", } Response: status: Created code: 201 js { "id": 2, "code": "fabric", "type": null, "readOnly": false, "order": 0, "description": { "id": 4, "language": "en", "name": "Fabric", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create option value

## Description:

API for the creation of a product option value. Product option value in this situation will be used to create product attribute with predefined lists of product option and product option value

## Method:

POST

## Url:

/api/v1/private/product/option/value

## Body:

```
js { "code": "coton", "descriptions": [ { "name": "Coton", "language": "en" }, { "name": "Coton", "language": "fr" } ], "order": 0 }
```

## Response:

status: Created

code: 201

js

```
{ "id": 1, "code": "coton", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 2, "language": "en", "name": "Coton", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create Product Property (With Option and predefined Option value)

## Description:

Create a product custom property using a predefined product option and a predefined product option value

## Method:

POST

## Url:

/api/v1/private/product/1/attribute

## Body:

```
{ "attributeDisplayOnly": true, "option": { "id": 2 }, "optionValue": { "id": 1 }, "sortOrder": 0 } Response: status: Created code: 201 js { "id": 1 }
```

## Create Product Property (With Option and no predefined Option value)

## Description:

Create a product custom property using a predefined product option and no predefined option value. That is any text you may want to add as an option value. As long as the value contains all defined languages.

## Method:

POST

## Url:

/api/v1/private/product/50/attribute

## Body:

```
{ }
```

```
{ "attributeDisplayOnly": true, "option": { "id": 2 }, "optionValue": { "descriptions": [ { "name":"Lin", "language":"fr" }, { "name":"Linen", "language":"en" } ] }, "sortOrder": 0 } Response: status: Created code: 201 js "id": 50
```

## List product properties

Description:

Method:

GET

## Url:

/api/v1/private/product/50/attributes

## Response: status: OK code: 200 js

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "attributes": [ { "id": 50, "sortOrder": 0, "attributeDefault": false, "attributeDisplayOnly": true, "productAttributeWeight": "0", "productAttributePrice": null, "productAttributeUnformattedPrice": null, "option": { "id": 2, "code": "fabric", "type": null, "readOnly": false, "order": 0, "description": { "id": 4, "language": "en", "name": "Fabric", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, "optionValue": { "id": 50, "code": "8ce3ae8f-6f05-4191-a635-3c76a42d439a", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 4,
```

```
"language": "en", "name": "Linen", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } } } ] }
```

## Get product property

Description: Method: GET Url: /api/v1/private/product/50/attribute/50 Response: status: OK

code: 200 js

```
{ "id": 50, "sortOrder": 0, "attributeDefault": false, "attributeDisplayOnly": true, "productAttributeWeight": "0", "productAttributePrice": null, "productAttributeUnformattedPrice": null, "option": { "id": 2, "code": "fabric", "type": null, "readOnly": false, "order": 0, "description": { "id": 4, "language": "en", "name": "Fabric", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, "optionValue": { "id": 50, "code": "8ce3ae8f-6f05-4191-a635-3c76a42d439a", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 0, "price": null, "description": { "id": 4, "language": "en", "name": "Linen", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null,
```

```
"metaDescription": null, "title": null } } }
```

## Remove a product property

## Description:

Removes a property from a product

## Method:

DELETE

## Url:

/api/v1/private/product/50/attribute/50

## Response:

status: OK

code: 200

js

## 15. api/products/product-search.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-search.html

## Product Search

Full text search of Products and Keywords autocomplete. Those apis requires to have a search starter such as Open Search starter configured.

## Search products

## Description:

Search products for a given Merchant Store and language.

## Method:

POST

## Url:

/api/v1/search

## Body:

```
js { "count": 20, "query": "shirt", "start": 0 }
```

## Response:

## status: OK

## code: 200

js

```
[ { "store": "default", "language": "en", "name": "Nike Sport Shirt", "description": "Nike sport shirt", "image": null, "addToCart": true, "brand": "Nike", "category": "Training Shirts", "variants": null, "attributes": null, "price": "34.00", "id": 26850, "reviews": null, "suggestions": null }, { "store": "default", "language": "en", "name": "T Shirt", "description": "Light cotton T Shirt", "image": "white-tshirt.jpeg", "addToCart": true, "brand": "DEFAULT", "category": "Cotton Shirts", "variants": [ { "size": "medium", "COLOUR": "white" } ], "attributes": { "fabric": "Coton" }, "price": "29.00", "id": 26750, "reviews": null, "suggestions": null } ]
```

## Search keywords

## Description:

Autocomplete search.

## Method:

POST

## Url:

/api/v1/search/autocomplete?store=DEFAULT&amp;language=en

## Body:

```
js { "count": 20, "query": "toto", "start": 0 } Response: status: OK code: 200 js "Nike Sport Shirt",
```

```
{ "values": [ "T Shirt" ] }
```

## 16. api/products/product-types.html

URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-types.html

## Product Types

Products can have a Product Type (type). A type can be used for grouping products based on specifics characteristics. A type can also be used at runtime to determine and drive code functionality..

## Create Product Type

## Description:

Creates a product type

## Method:

POST

## Url:

/api/v1/private/product/type

## Body:

js

```
{ "code" : "ecoproduct", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "Eco Product" }, { "language" : "fr", } ]
```

```
"name" : "Produit Ecologique" } Response: status: OK code: 200 js { "id": 50 }
```

## Edit Product Type

## Description:

Edit a product type

## Method:

PUT

## Url:

/api/v1/private/product/type/50

## Body:

```
{ "code" : "ecoproduct", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "Eco Product" }, { "language" : "fr", "name" : "Produit Écologique" } ] }
```

## Response:

status: OK

code: 200

js

## Get product type

## Description:

Get a product type by id

## Method:

GET

## Url:

/api/v1/private/product/type/50

## Body:

```
{ "code" : "ecoproduct", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "Eco Product" }, { "language" : "fr", } ]
```

```
"name" : "Produit Écologique" } Response: status: OK code: 200 js { "id": 50, "code": "ecoproduct", "visible": false, "allowAddToCart": false, "description": { "id": 2, "language": "en", "name": "Eco Product", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## List Product Types

## Description:

List product types by store. Request parameter stpre='CODE" can be used for filtering the store. If no request parameter is used then the system will use the user's associated store

## Method:

GET

## Url:

/api/v1/private/product/types

## Body:

```
js { "code" : "ecoproduct", "order" : 1, "descriptions" : [ { "language" : "en", "name" : "Eco Product" }, { "language" : "fr", "name" : "Produit Écologique" } ] }
```

## Response:

status: OK

code: 200

js

```
{ "totalPages": 1, "number": 0, "recordsTotal": 1, "recordsFiltered": 10, "list": [ { "id": 50, "code": "ecoproduct", "visible": false, "allowAddToCart": false, "description": { "id": 2, "language": "en", "name": "Eco Product", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } } ] }
```

## Delete Product Type

## Description:

Delete a product type by id

## Method:

DELETE

## Url:

/api/v1/private/product/type/50

## Response:

status: OK code: 200 js

## 17. api/products/product-variants.html

## URL: https://shopizer-ecommerce.github.io/documentation/api/products/product-variants.html

## Product Variants

Product variants will add the possibility to attach options to a product. Examples of product variants are: - White T Shirt - Medium Black T Shirt It essentially adds variations to a product definition. Requires the creation of Option then option Values

## Create an Option - color

## Description: Method:

POST

## Url:

/api/v1/private/product/option

## Body:

```
{ "code": "color", "descriptions": [ { "language" : "en", "name": "Color", "title": "Color" } ], "order": 10, "type": "select"
```

```
"description": "Color", } Response: status: Created code: 201 js { "id": 50, "code": "color", "type": "select", "readOnly": false, "order": 0, "description": { "id": 5, "language": "en", "name": "Color", "description": "Color", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create an Option Value

## Description:

```
Method: POST Url: /api/v1/private/product/option/value Body: js { "code": "white", "defaultValue": true, "descriptions": [ { "description": "White", "language": "en", "name": "White" } ], "sortOrder": 10 } Response: status: Created code: 201 js
```

```
{ "id": 100, "code": "white", "name": null, "defaultValue": false, "sortOrder": 0, "image": null, "order": 10, "price": null, "description": { "id": 5, "language": "en", "name": "White", "description": "White", "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }
```

## Create Product Variant

## Description:

Creates a Product Variant that contains an option and an option value.

## Method:

POST

## Url:

/api/v2/private/product/variant

## Body:

js

```
{ "code": "color-white", "defaultValue": true, "option": 50, "optionValue": 100, "sortOrder": 0 } Response: status: Created code: 201 js { "id": 0 } List Variants Description: List variants for a given Merchant Store Method: GET Url: /api/v2/private/product/variant?store=DEFAULT Response: status: OK code: 200 js
```

```
{ "totalPages": 1, "number": 1, "recordsTotal": 1, "recordsFiltered": 0, "items": [ { "id": 1, "code": "color-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 50, "code": "color", "type": "select", "readOnly": false, "name": "Color", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 100, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } } ] }
```

## Get Variant

Description:

Method:

Url:

/api/v2/private/product/variant/1

```
Response: status: OK code: 200 js { "id": 1, "code": "color-white", "date": null, "sortOrder": 0, "defaultValue": false, "option": { "id": 50, "code": "color", "type": "select", "readOnly": false, "name": "Color", "lang": "en", "variant": false, "optionValues": [] }, "optionValue": { "id": 100, "code": "white", "name": "White", "defaultValue": false, "sortOrder": 0, "image": null, "price": null, "description": null } }
```

## Remove Variant

## Description:

Removes a property from a product

## Method:

DELETE

## Url:

/api/v2/private/product/variant/1

Response: status: OK code: 200 js

## 18. api/products/product.html

## URL: https://shopizer-ecommerce.github.io/documentation/api/products/product.html

## Product

Product api is the complete object that should be used when listing products or getting product details. It differs from Product Definition since product definition is a summary of the product object.

## get product by sku

## Description:

Get a product by sku.

## Method:

GET

## Url:

/api/v2/product/79702984044?lang=en&amp;store=DEFAULT

## Response: status: OK code: 200 js

```
{ "id": 25655, "productShipeable": true, "available": true, "visible": true, "sortOrder": 1, "dateAvailable": "2022-03-28", "creationDate": "2022-03-28", "price": 1172, "quantity": 3, "sku": "79702984044", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": null, "weight": null, "length": null, "width": null, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": "79702984044", "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25244, "language": "en", "name": "Wrap-around handguard kit", "description": "<ul><li>Including all required assembly materials</ "friendlyUrl": "wrap-around-handguard-kit", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Wrap-around handguard kit" }, "productPrice": {
```

```
"id": 50558, "originalPrice": "SEK1,172.00", "finalPrice": "SEK1,172.00", "discounted": false, "description": { "id": 50372, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK1,172.00", "originalPrice": "SEK1,172.00", "discounted": false, "image": null, "images": [ { "id": 2103, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFAULT/7970 "externalUrl": "https://cms.jinomotos.com/pim/7/7b75de5b-c86b-4 "videoUrl": "https://cms.jinomotos.com/pim/7/7b75de5b-c86b-4ba8 "imageType": 1, "order": 0, "defaultImage": true }, { "id": 2104, "imageName": null, "imageUrl": "http://localhost:8080/static/products/DEFAULT/7970 "externalUrl": "https://cms.jinomotos.com/pim/2/285a6966-77b2-4 "videoUrl": "https://cms.jinomotos.com/pim/2/285a6966-77b2-43ae "imageType": 1, "order": 0, "defaultImage": false } ],
```

```
"manufacturer": null, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [ { "id": 10600, "code": "handguards", "sortOrder": 0, "visible": true, "featured": false, "lineage": "/10150//10004/10600/", "depth": 2, "parent": { "id": 10004, "code": "plastics" }, "description": { "id": 346, "language": "en", "name": "Hand Guards", "description": null, "friendlyUrl": "handguards", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Hand Guards" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true,
```

"owner": null }

## get product by friendly url

## Description:

Get a product by friendly url.

## Method:

GET

## Url:

/api/v2/product/slug/nike-sport-shirt

## Response: status: OK code: 200 js

```
{ "id": 26850, "productShipeable": true, "available": true, "visible": true, "sortOrder": 5, "dateAvailable": "2022-09-02", "creationDate": "2022-09-04", "price": 34, "quantity": 1000, "sku": "PRBSK01", "preOrder": false, "productVirtual": false, "quantityOrderMaximum": 1, "quantityOrderMinimum": 1, "productIsFree": false, "productSpecifications": { "height": 12, "weight": 250, "length": 8, "width": 14, "model": null, "manufacturer": null, "dimensionUnitOfMeasure": "cm", "weightUnitOfMeasure": "kg" }, "rating": 0, "ratingCount": 0, "refSku": null, "rentalDuration": 0, "rentalPeriod": 0, "description": { "id": 25628, "language": "en", "name": "Nike Sport Shirt", "description": "Nike sport shirt", "friendlyUrl": "nike-sport-shirt", "keyWords": "", "highlights": "", "metaDescription": "", "title": "Nike Sport Shirt" }, "productPrice": {
```

```
"id": 51900, "originalPrice": "SEK34.00", "finalPrice": "SEK34.00", "discounted": false, "description": { "id": 51111, "language": "en", "name": null, "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null, "priceAppender": null } }, "finalPrice": "SEK34.00", "originalPrice": "SEK34.00", "discounted": false, "image": null, "images": [], "manufacturer": { "id": 100, "code": "nike", "order": 0, "description": { "id": 32, "language": "en", "name": "Nike", "description": null, "friendlyUrl": null, "keyWords": null, "highlights": null, "metaDescription": null, "title": null } }, "attributes": [], "options": [], "variants": [], "properties": [], "categories": [
```

```
{ "id": 10800, "code": "sport-shirts", "sortOrder": 10, "visible": true, "featured": false, "lineage": "/10800/", "depth": 0, "parent": null, "description": { "id": 373, "language": "en", "name": "Training Shirts", "description": null, "friendlyUrl": "training-shirts", "keyWords": null, "highlights": null, "metaDescription": null, "title": "Training Shirts" }, "productCount": 0, "store": "DEFAULT", "children": [] } ], "type": { "id": 150, "code": "default", "visible": true, "allowAddToCart": true, "description": null }, "canBePurchased": true, "owner": null }
```

## 19. architecture/index.html

## Architecture

![Image 3](shopizer_image_003.jpeg)

Shopizer is enterprise open source e-commerce software for retailers who want flexibility, speed and control of their commerce platform. Shopizer is a software solution that gives organisations the ultimate flexibility to take an experience-first approach to commerce, with simple powerful APIs and built in store models.

## Key technical benefits

- Headless commerce
- Services built with Spring framework (Spring boot, Spring Security)
- Front end Angular - React - Vuejs
- CICD, Vulnerability checks
- Open source and open standards
- On Prem - Cloud
- Run from containers or from individual processes (npm, tomcat)
- Extensions are built as spring starter components

![Image 4](shopizer_image_004.jpeg)

## Integrationss

Shopizer supports integration with external payment and shipping modules such as Stripe, Fedex, Braintree, Square USPS and more. A set of module extensions allows using various content management storage such as JBoss Infinispan, AWS S3 and external web servers such as NGINX or Apache server. Elastic and Open Searh provide searching functionality.

A REST api exposes all commerce functionality (B2C, B2B, C2C, Multi-Stores) as well as complete system administration. Spring Security configured out of the box with JWT Bearer token authentication provides application interface authentication and authorization.

## 20. build-and-run.html

URL: https://shopizer-ecommerce.github.io/documentation/build-and-run.html

## Build and run Shopizer

The purpose of this section is to provide new users with the basics of Shopizer e-commerce framework in order to get started in their projects. It will walk the reader through the steps of installing and configuring Shopizer as well as running demo application locally. These instructions are for running legacy Shopizer with JSP frontends as well as Shopizer backend version with Angular admin and React shop front.

## Build Shopizer

![Image 5](shopizer_image_005.jpeg)

Build from the source files. Open a terminal or console sh git clone git@github.com:shopizer-ecommerce/shopizer.git cd shopizer

## Build using Maven

sh mvnw clean install This command should result in a success message. If you have any errors during this process and require assistance,feel free to use Shopizer forum. See Shopizer forum for asking or searching Shopizer build related questions.

## Start Shopizer

sh cd sm-shop mvnw spring-boot:run

Once the terminal or console displays that Shopizer is running and listening on service port and ready to be used.

Open a browser and type url localhost:8080

This will open Shopizer headless api. You can view all application interfaces available by opening localhost:8080/swagger-ui.html

## Build and run Shopizer Angular administration app

![Image 6](shopizer_image_006.jpeg)

Angular app consuming Shopizer administration services endpoints sh

git clone git@github.com:shopizer-ecommerce/shopizer-admin.git cd shopizer-admin

## Build using npm

sh npm i

## Start app

sh ng serve -o

Open a browser on localhost:4200

Username:

admin@shopizer.com

Password:

password

All software requirements for Node and Angular are specified in README.md file int the project root

## Build Shopizer store React app

![Image 7](shopizer_image_007.jpeg)

This is a showcase React app consuming Shopizer services endpoints From a terminal or console sh git clone git@github.com:shopizer-ecommerce/shopizer-shop-reactjs.git cd shopizer-shop-reactjs

## Build using npm

sh npm i

This command should result in a success message. If you have any errors during this process and require assistance,feel free to use Shopizer forum. See Shopizer forum for asking or searching Shopizer build related questions.

## Start app

sh npm run dev Once the terminal or console displays that Shopizer is running and listening on service port and ready to be used.

## All software requirements for Node js are specified in README.md file int the project root

## 21. configuration/database.html

URL: https://shopizer-ecommerce.github.io/documentation/configuration/database.html

## Database configuration

Shopizer has been tested and supports H2, MySQL (recommended), MariaDB and Postgresql databases. Shopizer comes pre-configured with H2 database. H2 is an embedded database (http://www.h2database.com) configured by default to save data on files.

This pre-configuration should only be used for testing and never be used in a production environment.

## Configure Shopizer with H2 database

By default H2 files are saved into your application server runtime directory. To change the location where the files have to be saved, edit sm-shop/src/main/resources/database.properties and edit the line sh

```
db.jdbcUrl=jdbc\:h2\:file\:c:/SALESMANAGER;AUTOCOMMIT\=OFF;INIT\=CREATE SCH
```

Make sure the file path is a valid existing directory. In this case the configuration file will be saved in c:\

## Configure Shopizer with MySQL

Log to your MySQL as root and create schema SALESMANAGER

```
sh mysql>CREATE DATABASE SALESMANAGER; mysql>GRANT USAGE, SELECT ON *.* TO <YOUR USERNAME>@localhost IDENTIFIED BY mysql>GRANT ALL ON SALESMANAGER.* TO <YOUR USERNAME>@localhost; mysql>GRANT FILE ON *.* TO <YOUR USERNAME>@localhost; mysql>FLUSH PRIVILEGES;
```

## Example

mysql&gt;CREATE DATABASE SALESMANAGER; mysql&gt;GRANT USAGE, SELECT ON *.* TO testuser@localhost IDENTIFIED BY 'passw mysql&gt;GRANT ALL ON SALESMANAGER.* TO testuser@localhost; mysql&gt;FLUSH PRIVILEGES;

## Create user that can connect from everywhere

```
mysql>CREATE USER 'mysuer'@'%' IDENTIFIED BY 'mypassword'; mysql>GRANT USAGE, SELECT ON MYDB.* TO 'mysuer'@'%' with grant option; mysql>GRANT ALL ON MYDB.* TO 'mysuer'@'%';
```

```
sh mysql>FLUSH PRIVILEGES;
```

## Edit sm-shop/src/main/resources/database.properties

sh

```
db.jdbcUrl=jdbc:mysql://localhost:3306/SALESMANAGER?autoReconnect=true&useU db.user=YOUR USERNAME db.password=YOUR PASSWORD db.driverClass=com.mysql.jdbc.Driver hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect db.preferredTestQuery=SELECT 1
```

db.schema=SALESMANAGER hibernate.hbm2ddl.auto=update

Note: Database and table names are not case sensitive in Windows, and case sensitive in most varieties of Unix-Linux. Consequently MySQL which uses lower case schema and table names will not be able to see the schema and tables which are created upper case in Shopizer. If you get an exception of a table not being found and being displayed in lower case in the error message, you will have to specify this property in My.cnf. To resolve the issue set the lower\_case\_table\_names to 1 in My.cnf configuration file.

## Edit My.cnf

sh lower\_case\_table\_names=1

## Configure Shopizer with Postgresql

```
Create a new postgresql sh CREATE DATABASE SHOPIZER; Create schema sh CREATE SCHEMA salesmanager; Edit sm-shop/src/main/resources/database.properties sh #POSTGRES db.jdbcUrl=jdbc:postgresql://localhost:5432/SHOPIZER db.user=postgres db.password=password db.driverClass=org.postgresql.Driver hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect db.preferredTestQuery=SELECT 1 db.schema=SALESMANAGER hibernate.hbm2ddl.auto=update Edit shopizer/pom.xml Un-comment both postgresql dependency sh <!--  postgres <dependency> <groupId>org.postgresql</groupId> <artifactId>postgresql</artifactId> </dependency> -->
```

## 22. configuration/emails.html

URL: https://shopizer-ecommerce.github.io/documentation/configuration/emails.html

## Emails configuration

Shopizer supports 2 methods for sending emails:

- smtp ( default )
- AWS Simple Email Service ( ses )

## SMTP Configuration (Default method)

This can be done by configuring SMTP connectivity at the server level from Shopizer administration console. When logging in the console move to Configurations and then to Email settings. SMTP configuration uses underlying Spring SpringHtmlEmailSender for sending HTML based emails.

Protocol : SMTP or SMTPS (this depends on SMTP provider. It usualy is SMTPS) Host : SMTP server host Post : Most of the time either 25 or 465 Username : SMTP server attributed username Password : SMTP server attributed password Requires authentication : Should always be true TLS : Should always be true (Usualy mentioned by SMTP provider)

## Shopizer configuration

Make sure shopizer-core.properties in sm-core project is configured fot smtp method before starting the server.

sh config.emailSender=default

## SES (AWS Simple Email SDK)

This method uses AWS SDK implementation to perform SMTP(S) connectivity. AWS SES SDK is already included in Shopizer Maven POM file.

```
<!-- https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk-ses --> <dependency> <groupId>com.amazonaws</groupId> <artifactId>aws-java-sdk-ses</artifactId> </dependency>
```

## Shopizer configuration

Make sure shopizer-core.properties in sm-core project is configured fot ses method before starting the server. It requires 2 specific property for connecting to AWS Simple Email Service (ses method and region in which ses is configured in your AWS account)

sh

```
config.emailSender=ses config.emailSender.region=US_EAST_1
```

## 23. docker.html

URL: https://shopizer-ecommerce.github.io/documentation/docker.html

## Run Shopizer with Docker Compose

![Image 8](shopizer_image_008.jpeg)

Shopizer can be run from Docker containers. The following instructions will run Shopizer Headless, Shopizer Shop App, Shopizer Admin App and Mysql sh

git clone git@github.com:shopizer-ecommerce/shopizer-docker-compose.git cd shopizer docker compose

Now run Docker containers

Open a browser on this urls

| Application   | Url            |
|---------------|----------------|
| Headless      | localhost:8080 |
| Shop          | localhost:3000 |
| Admin         | localhost:4200 |

Admin app credentials

Username:

admin@shopizer.com

Password:

password

## 24. model/country-zones.html

URL: https://shopizer-ecommerce.github.io/documentation/model/country-zones.html

## Country and Zones

## Country codes

Country codes are defined in shopizer sm-core SchemaConstants (com.salesmanager.core.constants.SchemaConstants)

sh

```
public static final String[] COUNTRY_ISO_CODE = { "AF","AX","AL","DZ", "AS","AD","AO","AI","AQ","AG","AR","AM","AW","AU","AT","AZ" "BD","BB","BY","BE","BZ","BJ","BM","BT","BO","BA","BW","BV" "BN","BG","BF","BI","KH","CM","CA","CV","KY","CF","TD","CL" "CC","CO","KM","CG","CD","CK","CR","CI","HR","CU","CY","CZ" "DM","DO","EC","EG","SV","GQ","ER","EE","ET","FK","FO","FJ" "GF","PF","TF","GA","GM","GE","DE","GH","GI","GR","GL","GD" "GT","GG","GN","GW","GY","HT","HM","VA","HN","HK","HU","IS" "IR","IQ","IE","IM","IL","IT","JM","JP","JE","JO","KZ","KE" "KR","KW","KG","LA","LV","LB","LS","LR","LY","LI","LT","LU" "MG","MW","MY","MV","ML","MT","MH","MQ","MR","MU","YT","MX" "MC","MN","ME","MS","MA","MZ","MM","NA","NR","NP","NL","AN" "NI","NE","NG","NU","NF","MP","NO","OM","PK","PW","PS","PA" "PE","PH","PN","PL","PT","PR","QA","RE","RO","RU","RW","SH" "PM","VC","WS","SM","ST","SA","SN","RS","SC","SL","SG","SK" "SO","ZA","GS","ES","LK","SD","SR","SJ","SZ","SE","CH","SY" "TZ","TH","TL","TG","TK","TO","TT","TN","TR","TM","TC","TV" "AE","GB","US","UM","UY","UZ","VU","VE","VN","VG","VI","WF" "YE","ZM","ZW" };
```

Country description are derived from java Locale class and then saved to CountryDescription table

## Zones List

Zones are loaded from sm-core/src/main/resources/zoneconfig.json file which contains zones codes and names per country and language. This file is loaded during initial startup and is performed only once during the application life cycle.

![Image 9](shopizer_image_009.jpeg)

## 25. model/merchant.html

URL: https://shopizer-ecommerce.github.io/documentation/model/merchant.html

## Merchant model

This section explains possible possible configurations for MerchantStore in order to use the app as a single store application, multiple store application (marketplace) or retailer - store application.

![Image 10](shopizer_image_010.jpeg)

## Single store

Simple stores are unique stores managing their own products and inventory. This type of store is configured by default in Shopizer and is created with unique store code DEFAULT. Front end can be served by default jsp shop or Angular and ReactJS examples

## Multiple stores

Multiple independent stores can be managed in Shopizer. Each store has its own configuration and each store manages their own inventory, products, category and content. Each store requires their own shop that can be run on independent processes. Default jsp, Angular and React store fronts can be used as different processes to run each store.

## Schema

## MERCHANT\_STORE****MERCHANT\_LANGUAGE

A MERCHANT (vendor) can be standalone for single stores use cases. A MERCHANT can have a parent for franchiser / franchisee use cases.

A MERCHANT can be support one to multiple languages

![Image 11](shopizer_image_011.jpeg)

## 26. model/product-references.html

URL: https://shopizer-ecommerce.github.io/documentation/model/product-references.html

## Product Reference model

This section explains the related entity of a Product item. A Product can be categoryzed and typed. A Product has also a Brand (Manufacturer or Collection) associated to it.

![Image 12](shopizer_image_012.jpeg)

## 27. model/user.html

URL:

https://shopizer-ecommerce.github.io/documentation/model/user.html

## User model

## Business rules

| Role         | Description                                              |
|--------------|----------------------------------------------------------|
| superadmin   | Can create, modify any user of any stores                |
| admin        | Can create, modify any user from its pertaining store    |
| admin_store  | Can create, modify any user from its pertaining store    |
| admin_retail | Can only modify their profile, Deletion is not supported |

## Global Rules

- User even if superuser can't change roles
- Superuser can't create superuser
- Admins can't delete or modify superuser

## Pages rules

## User List

- When login in as super admin all users are displayed in the list
- When login in a any other admin only users from same store are displayed in the list
- User cannot delete himself, he has to rely on other admin

## Create / modify user page

- Cannot assign group to superadmin
- Self cannot edit it's own groups
- Self cannot change merchant store
- Default selected merchant is current user merchant

## Schema

## USERMERCHANT\_STORESM\_GROUPUSER\_GROUPPERMISSION\_GROUP****PERMISSION

Users belong to MERCHANT\_STORE. Users are assigned to groups having predefined sets of permissions.

![Image 13](shopizer_image_013.jpeg)

![Image 14](shopizer_image_014.jpeg)

## 28. tools/translation.html

URL: https://shopizer-ecommerce.github.io/documentation/tools/translation.html

## Shopizer language pack generator

This software generates Shopizer labels for a new language. Shopizer comes with a few language packs already created such as english and frech; when a new language is required 2 options are possible (1) copy and create languages by hand or (2) use this software to auto-generate a new one.

![Image 15](shopizer_image_015.jpeg)

## requirements

-- Most recent version of Shopizer on your machine -- An AWS (Amazon Web Service) account -- Java SDK (version 1.8 +)

## prepare instructions

- Clone Shopizer from github

sh git clone -b master https://github.com/shopizer-ecommerce/shopizer.git

- Look in shopizer/sm-shop/src/main/resources/bundles if language files are already there for required language (files have this format file\_&lt;2 letter language iso code&gt;.properties (ex shopizer\_fr.properties)

sh cd LanguagePackBuilder mvnw clean install

- Execute generate language
- Requires shopizer directory absolute path (1)
- Requires target language 2 letter iso ced (example es for Spanish) (2)
- Requires AWS access key (3)
- Requires AWS secret access key (4)
- parameters 1 and 2 to be sent as execution parameters and parameters 3 and 4 to be sent as environment variables

sh

```
cd target java -jar LanguagePackBuilder-1.0.0-SNAPSHOT.jar com.shopizer.tools.languag
```

## Share your new language with us !

- Sanitize and 2x check results before sharing
- Open a pull request in Shopizer
- Add role to AWS user
- Log on to AWS and add 'TranslateFullAccess' to your favorite user. See this document for more details https://docs.aws.amazon.com/IAM/latest/UserGuide/id\_roles\_create\_for-user.html
- Get your AWS access key and secret access key (more details here https://aws.amazon.com/premiumsupport/knowledge-center/create-access-key/)

## generate language instructions

- Clone Language software from github

sh

```
git clone -b master https://github.com/shopizer-ecommerce/languages.git
```

- Build language application

## 29. what-is-shopizer.html

## URL: https://shopizer-ecommerce.github.io/documentation/what-is-shopizer.html

## What is Shopizer

![Image 16](shopizer_image_016.jpeg)

Shopizer is enterprise open source e-commerce software for retailers who want flexibility, speed and control of their commerce platform. Shopizer is a software solution that gives organizations the ultimate flexibility to take an experience-first approach to commerce, with simple powerful APIs and built in stores models. Key technical benefits

- Built with Spring framework
- Vulnerability checks
- Open source and open standards
- Cloud Ready: Deploy Shopizer in the public
- or private cloud on Amazon Webservices (AWS), Microsoft
- Azure or Google Cloud Platform (GCP)
- Run on premise servers
- Run from Docker containers
- Run from any Java servlet containers such as Tomcat

Shopizer is a an application composed of a set of services written in Java that serves functionality needed to build entreprise e-commerce systems. The software can be used out of the box as a complete ready to use web application including a few web stores templates you can choose from. Shopizer can also be used as a restful backend application for serving a whole set of commerce REST apis on which you can build your own tailored e-commerce front store. Shopizer e-commerce system provides the following functionality:

- Catalog and products management
- Shopping cart
- Content management
- Marketing components
- Smart pricing
- Ordering
- Payment
- Shipping

| Scope                                      | Features                     |
|--------------------------------------------|------------------------------|
| Software                                   | Supports MySQL               |
| Supports Postgresql                        |                              |
| Supports MariaDB                           |                              |
| Supports H2                                |                              |
| Java - Spring Boot                         |                              |
| RESTAPI                                    |                              |
| Cloud centric                              |                              |
| Container based                            |                              |
| Open source Apache v2 licence              |                              |
| Catalog management                         | SKU based product management |
| One to many prices per item                |                              |
| Simple time based promotion                |                              |
| Complex promotions (Based on JBoss Drools) |                              |
| Category hierarchy                         |                              |
| Product options                            |                              |
| Product variants                           |                              |
| Custom product properties                  |                              |
| B2C                                        |                              |
| B2B                                        |                              |
| C2C                                        |                              |

Scope

Features

Multiple stores

Promotions

Promotions applicable at item level

Promotions applicable at cart level

Customer management

Self serve customer entry point

Social authentication

Customer registration

Rule based engine

Multiple currency

Ordering

Multiple payment modules

Multiple shipping modules

Promotions based on shipping rules

Promotions based vatious property (quantity of items, customer...)

REST API

Administration api

Products api

Customer api

Order api

Cart api

Shipping api

Payment api

User api

Search

Search

Based on Elastic Search and Open Search

Search items

| Scope               | Features                                              |
|---------------------|-------------------------------------------------------|
| Search autocomplete |                                                       |
| Languages           | Our language translator converts to multiple language |