# Using REST resources

To call the Admin REST API, you can use the [REST client](../reference/clients/Rest.md) to send manually crafted requests to Shopify.

The Admin API has a lot of endpoints, and the differences between them can be subtle.
To make it easier to interact with the API, this library provides resource classes, which map these endpoints to OO code and can make API queries feel more natural.

> **Note**: we provide auto-generated resources for all **_stable_** API versions.
> If your app is using `unstable` or a Release Candidate, you can still import REST resources (see [mounting REST resources](#mounting-rest-resources) below) for other versions, but we'll log a warning to remind you to update when you're ready.

Below is an example of how REST resources can make it easier to fetch the first product and update it:

<div>With a plain REST client:

```ts
// App must provide response types
interface ProductResponse {
  product: {
    id: number;
    title: string;
    // ...
  };
}

const sessionId = await shopify.session.getCurrentId({
  rawRequest: req,
  rawResponse: res,
});

// use sessionId to retrieve session from app's session storage
// getSessionFromStorage() must be provided by application
const session = await getSessionFromStorage(sessionId);

const client = new shopify.clients.Rest({session});

// The following line sends a HTTP GET request to this constructed URL:
// https://${session.shop}/admin/api/${shopify.config.api_version}/products/7504536535062.json
const response = await client.get<ProductResponse>({
  path: 'products/7504536535062',
});

// Apps needs to dig into the response body to find the object
response.body.product.title = 'A new title';

// The following sends a HTTP PUT request to this constructed URL...
// https://${session.shop}/admin/api/${shopify.config.api_version}/products/${response.body.product.id}.json
// ... with this body
// {"product":{"id":response.body.product.id,"title":"A new title"}}
await client.put({
  path: `products/${response.body.product.id}`,
  data: response.body.product,
});
```

</div><div>With REST resources:

```ts
const sessionId = await shopify.session.getCurrentId({
  rawRequest: req,
  rawResponse: res,
});

// use sessionId to retrieve session from app's session storage
// getSessionFromStorage() must be provided by application
const session = await getSessionFromStorage(sessionId);

// get a single product via its product id
const product = await shopify.rest.Product.find({session, id: '7504536535062'});

product.title = 'A new title';

await product.save({
  update: true,
});
```

</div>

The resource classes provide representations of all endpoints for the API resource they cover. A few examples:

1. `GET /products/{product_id}.json` maps to `Product.find()`
1. `POST /products.json` maps to `product.save()` (as an instance method)
1. `GET /products.json` maps to `Product.all()`
1. `GET /products/count.json` maps to `Product.count()`

Please visit our [REST API reference documentation](https://shopify.dev/api/admin-rest) for detailed instructions on how to call each of the endpoints.

## Mounting REST resources

The library doesn't include all available resources automatically to make loading it more efficient, since there are hundreds of resource classes.
To use REST resources, you can import the desired version and mount it onto your `shopify` object when creating it:

```ts
import {shopifyApi} from '@shopify/shopify-api';
import {restResources} from '@shopify/shopify-api/rest/admin/2022-10';

const shopify = shopifyApi({
  ...,
  apiVersion: ApiVersion.October22,
  restResources,
});
```

This will automatically load all REST resources onto `shopify.rest`, as per the example above.
From this point, you can start using the resources to interact with the API.

## Paginated requests

Shopify's REST API supports [cursor-based pagination](https://shopify.dev/api/usage/pagination-rest), to limit the amount of data sent to an app on a single request.

Each request will return the information required for an app to request the previous / next set of items.

For REST resources, the class will contain the information necessary to make those requests in the `NEXT_PAGE_INFO` and `PREV_PAGE_INFO` properties.
These values will always reflect the last request made with that class.

Here is an example for fetching more than one set of products from the API:

```ts
do {
  const pageProducts = await shopify.rest.Product.all({
    ...shopify.rest.Product.NEXT_PAGE_INFO?.query ?? {
      status: 'active',
    },
    session,
  });

  // ... use pageProducts
} while (shopify.rest.Product.NEXT_PAGE_INFO);
```

> **Note**: these properties are not thread-safe because they're stored statically in the class.
> If you are using this feature to send requests in parallel, you can store the `query` property in a thread-safe way, since it's a plain JavaScript object.

[Back to guide index](../../README.md#guides)
