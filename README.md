# OpenAPI Operation Executor

This library provides a helper method to perform operations (web requests) specified by an [OpenAPI spec](https://www.openapis.org/).

Every operation in an OpenAPI spec has a unique field used to identify it, called an `operationId`. This library allows you to send properly formatted requests to an API by supplying an OpenAPI spec (as JSON), the `operationId` of the operation you want to perform, and the arguments for that operation. This makes it so that you do not have to take the time to generate an SDK out of an OpenAPI spec every time you need to work with a new spec or everytime an API/spec changes (for example using the [openapi npm package](https://github.com/openapi/openapi)). This also reduces your application's bundle size by removing dependencies on multiple SDKs per API your application needs to interact with.

## Installation

```shell
npm install https://github.com/semanticsAnonymous/openapi-operation-executor.git
```

or

```shell
yarn add https://github.com/semanticsAnonymous/openapi-operation-executor.git
```

## Usage

Using Typescript:

```ts
import { OpenApiOperationExecutor } from "openapi-operation-executor";
import type { OpenApi } from "openapi-operation-executor";

// Import your OpenAPI spec as a JSON module (requires the resolveJsonModule flag in typescript).
// Alternatively, you could read a YAML file uing fs and use the yaml npm module to convert to JSON.
import openApiSpec from "./path/to/openapi-spec.json";

// Initialize the OpenAPI Operation Executor
const executor = new OpenApiOperationExecutor();
await executor.setOpenapiSpec(openApiSpec as OpenApi);

// Execute the operation and get the response
const response = await executor.executeOperation(
  "operationId",
  { accessToken: "YOUR_ACCESS_TOKEN" },
  { arg: "arg-value" }
);
```

## Browser support

To use OpenApiOperationExecutor in a browser, you'll need to use a bundling tool such as Webpack, Rollup, Parcel, or Browserify. Some bundlers may require a bit of configuration, such as setting `browser: true` in rollup-plugin-resolve.

## API

### executeOperation

The `executeOperation` method of an `OpenApiOperationExecutor` instance sends a properly formatted web request to the API described in the provided OpenAPI spec (as JSON). This library uses [axios](https://github.com/axios/axios) to send web requests.

Requests automatically use the method (GET, POST, etc.) that the OpenAPI operation is nested under in the provided spec.

⚠️ This library currently only supports sending JSON data in request bodies. It automatically adds the header `Content-Type: application/json` to every request it sends.

**Parameters**

| Parameter       | Type     | Required | Description                                                                                                                                                                                                                                        |
| :-------------- | :------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operationId`   | `string` | Required | The operationId of the operation to perform.                                                                                                                                                                                                       |
| `configuration` | `object` | Required | An `OpenApiClientConfiguration` object.                                                                                                                                                                                                            |
| `args`          | `any`    |          | Data conformant to the specified `requestBody` of the OpenAPI operation being performed to be send to the API as the body of the request. It will be serialized according to the `Content-Type` header, which is always set to `application/json`. |
| `options`       | `object` |          | An `AxiosRequestConfig` object to override any request configurations. See [the axios API documentation](https://github.com/axios/axios#request-config) for reference.                                                                             |

**Configuration**

These are the available config options for making requests (in Typescript):

```ts
export interface OpenApiClientConfiguration {
  /**
   * Parameter for HTTP authentication with the Bearer security scheme
   */
  bearerToken?:
    | string
    | Promise<string>
    | (() => string)
    | (() => Promise<string>);
  /**
   * Parameter for apiKey security where the key is the name
   * of the api key to be added to the header or query. Multiple api keys
   * may be provided by name.
   * @param name - security name
   */
  [key: string]:
    | undefined
    | string
    | Promise<string>
    | ((name: string) => string)
    | ((name: string) => Promise<string>);
  /**
   * Parameter for apiKey security which will be used if no named api key
   * matching the required security scheme is supplied.
   * @param name - security name
   */
  apiKey?:
    | string
    | Promise<string>
    | ((name: string) => string)
    | ((name: string) => Promise<string>);
  /**
   * Parameter for HTTP authentication with the Basic security scheme
   */
  username?: string;
  /**
   * Parameter for HTTP authentication with the Basic security scheme
   */
  password?: string;
  /**
   * Parameter for oauth2 security
   * @param name - security name
   * @param scopes - oauth2 scope
   */
  accessToken?:
    | string
    | Promise<string>
    | ((name?: string, scopes?: string[]) => string)
    | ((name?: string, scopes?: string[]) => Promise<string>);
  /**
   * Override base path
   */
  basePath?: string;
  /**
   * Base options for axios calls
   */
  baseOptions?: any;
}
```

⚠️ This library currently supports OpenApi security types `oauth2`, `apiKey`, and `http` (with scheme `basic` or `bearer`). See [the OpenApi Spec](https://spec.openapis.org/oas/v3.1.0#security-scheme-object) for reference.

- When using `oauth2` type security, if an `accessToken` string or function is supplied, an `Authorization` header will be added with the value `Bearer <accessToken>`.
- When using `apiKey` type security, if an `apiKey` string or function supplied, the return value will be added to the header or query of the request depending on the OpenAPI spec's Security Schemes. This library does not work with apiKeys as cookies.
- When using `http` security with the `basic` scheme, if a `username` and `password` are supplied, an `Authorization` header will be added with the value `Basic {CREDENTIALS}` where `CREDENTIALS` is the username and password concatenated with a colon character ":" and base64 encoded as defined in the spec ([RFC7617](https://www.rfc-editor.org/rfc/rfc7617.html)).
- When using `http` security with the `bearer` scheme, if a `bearerToken` string or function is supplied, an `Authorization` header will be added with the value `Bearer {bearerToken}` as deinfed in the spec ([RFC6750](https://www.iana.org/go/rfc6750)).
- `mutualTLS` and `openIdConnect` security types are not yet supported.

**Return value**

`executeOperation` returns a [`Promise`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) which resolves to an [`AxiosResponse`](https://github.com/axios/axios#response-schema).

Access the response using Promise syntax:

```ts
executor
  .executeOperation("operationId", { accessToken: "ACCESS_TOKEN" })
  .then((response) => {
    // Do something with the response...
  })
  .catch((error) => {
    // Handle an error...
  });
```

Access the response using `async`/ `await` syntax:

```ts
const response = await executor.executeOperation("operationId", {
  accessToken: "ACCESS_TOKEN",
});
```

These are the fields exepcted in an [`AxiosResponse`](https://github.com/axios/axios#response-schema) (in Typescript):

```ts
export interface AxiosResponse<T = any, D = any> {
  // The response that was provided by the server
  data: T;
  // The HTTP status code from the server response
  status: number;
  // The HTTP status message from the server response
  statusText: string;
  // `headers` the HTTP headers that the server responded with
  // All header names are lowercase and can be accessed using the bracket notation.
  // Example: `response.headers['content-type']`
  headers: AxiosResponseHeaders;
  // The config that was provided to `axios` for the request
  config: AxiosRequestConfig<D>;
  // `request` is the request that generated this response
  // It is the last ClientRequest instance in node.js (in redirects)
  // and an XMLHttpRequest instance in the browser
  request?: any;
}
```
