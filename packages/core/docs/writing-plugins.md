---
priority: 4
title: Writing plugins
---

Plugins are the strength of Phenomic and what make it so flexible and reusable.
Phenomic allows you to interact with it at several state of the process
depending on what you need.

[Learn more about Phenomic lifecycle](./how-phenomic-works.md#lifecycle)

To know how to use a plugin see
[the configuration `plugins` option](./configuration.md#plugins).

## Creating your plugin

A plugin can take the form of a node_modules or directly a function in the
configuration. It must be a function that returns object.

```js
module.exports = (/* options */) => {
  return {
    // plugin
    name: "phenomic-plugin-name",

    // fake method name
    method1: () => {},
    method2: () => {},
  };
};
```

Note that you can also export using ES2015 export

```js
const yourPlugin = (/* options */) => {
  return {
    // plugin
    // ...
  };
};

export default yourPlugin;
```

If your plugin accepts options, you can use the first parameter of the function.
This parameter will receive the option that you can inject via
[the configuration `plugins` option](./configuration.md#plugins).

## Examples

The best source of examples will be phenomic
[plugins included in our repository](https://github.com/phenomic/phenomic/tree/master/packages).

Plugins interesting part are usually located at
`phenomic/packages/plugin-*/src/index.js`.

## Lifecycle methods

Here are the list of methods that allows you to interact with Phenomic
lifecycle.

### `collect`

The method `collect` is useful if you plan to use [Content API](./api.md). It
allows you to collect your data before rendering and to inject it into the
database.

```js
collect: ({ db, transformers }) => Promise;
```

It receive the database and all transformers plugin as arguments

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-collector-files/src/index.js

### `supportedFileTypes` + `transform`

The method `transform` is useful if you plan to use [Content API](./api.md). It
allows you to transform files (that match `supportedFileTypes` - array of files
extensions) as something ready for Phenomic database.

#### `supportedFileTypes`

Array of files extensions.

Example

```js
supportedFileTypes: ["md", "markdown"]; // will match *.md and *.markdown
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-markdown/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-asciidoc/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-json/src/index.js

#### `transform`

Function that accepts an object and must return an result ready for Phenomic
database:

```js
transform: ({
  // you will only receive file that match `supportedFileTypes`
  file: { name: string, fullpath: string },
  contents: Buffer
}) {
  // do your thing to parse `contents`
  // ...

  // then return a database entry
  return {
    data: Object,
    partial: Object
  };
}
```

Learn more about database entries in [Content API documentation](./api.md).

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-markdown/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-asciidoc/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-transform-json/src/index.js

### `build`

Allows you to do a JavaScript bundle for client side rendering (CSR). Should
return an object (map) of files (js and css) generated by the bundler.

```js
build: () => assets,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-bundler-webpack/src/index.js

### `buildForPrerendering`

Allows you to do a JavaScript bundle for static side rendering (SSR).

The result will be passed to the [`getRoutes()`](#getroutes) method. Your can
return a promise.

```js
buildForPrerendering: () => Promise<result that will be passed to `getRoutes`>,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-bundler-webpack/src/index.js

### `getRoutes`

```js
getRoutes: (resultOf_buildForPrerendering) => routes,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-renderer-react/src/index.js

### `resolveURLs`

Allows to resolve all possibles URLS for static side rendering (SSR). Receive an
object with the routes returned by `getRoutes`.

```js
resolveURLs: ({
  routes
}) => Promise<Array<url>>,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-renderer-react/src/index.js

### `renderStatic`

Allows to statically generated some files. Receive an object that contains the
result of `buildForPrerendering`, `build` and the url to render. Should return
an array of files (a file is a `path` + its `contents`). Promise are accepted.

```js
renderStatic: ({|
  app /* result of `buildForPrerendering` */,
  assets /* result of `build` */,
  location /* url to prerender */
|}) => Promise<Array<{ path, contents }>>,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-renderer-react/src/index.js

### `renderDevServer`

Allows to generated the HTML for the entry point for development. Receive an
object that contains the result of `build` and the url to render.

Should return some HTML.

```js
renderDevServer: ({
  assets  /* result of `build` */,
  location /* url to prerender */
}) => htmlString,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-renderer-react/src/index.js

### `addDevServerMiddlewares`

Allows you to inject some express middleware for development server. Should
return an array of express middlewares. Promise are accepted.

```js
addDevServerMiddlewares: () =>
  | Array<express$Middleware>
  | Promise<Array<express$Middleware>>,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-bundler-webpack/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-public-assets/src/index.js
- https://github.com/phenomic/phenomic/tree/master/packages/plugin-rss-feed/src/index.js

### `beforeBuild`

Allows you to do anything before the build. Can return a promise.

```js
beforeBuild: () => void | Promise<void>,
```

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-public-assets/src/index.js

### `afterBuild`

Allows you to do anything after the build. Can return a promise.

```js
afterBuild: () => void | Promise<void>
```

### `extendAPI`

Allows to extend the API with new endpoints. Handy for custom queries to the
database. Accept an object with API (express server) and Phenomic databse to hit
directly on it.

Example

```js
extendAPI: ({
  apiServer /* express api server */,
  db /* phenomic databse */
}) => {
  apiServer.get("/your-custom-path/:path/:id.json", async function(
    req,
    res
  ) {
    try {
      // get an item from its id
      const entry = db.get(req.params.path, req.params.id);

      // some business logic
      // ...

      res.json(/* your json */);
    } catch (error) {
      res.status(404).end();
    }
  });
},
```

Learn more about database [Content API documentation](./api.md).

Used in

- https://github.com/phenomic/phenomic/tree/master/packages/plugin-api-related-content/src/index.js
