---
title: "Authoring TypeScript Libraries"
date: "2018-03-16"
template: "post"
draft: false
slug: "/posts/authoring-typescript-libraries/"
category: "TypeScript"
tags:
  - "TypeScript"
description: "You've written a TypeScript library. Now what? Time to publish, of course. But how? Do you publish it as a TypeScript module or compile it to JavaScript? Should it run in browser or Node.js, or both? What about ES5  versus ES6 or ESNext? And don't forget, some people prefer AMD modules."
---

## Problem

You've written a TypeScript library. Now what? Time to publish, of course. But how? Do you publish it as a TypeScript module or compile it to JavaScript? Should it run in browser or Node.js? Or both? What about ES5 versus ES6 or ESNext? And don't forget, some people prefer AMD modules.

In this article, I address all of these concerns and more. I will show you how to publish a TypeScript library to a broad audience. Publishing a TypeScript library can be complicated, but it doesn't have to be.

## TLDR

Use tsc to compile TypeScript source code to ES modules. Use webpack + babel to transpile ES modules to a umd package. Providing ES modules and a umd package supports a larger developer community.

---

## Solution

A two-step build process.

1. Compile TypeScript source code to ES modules using `tsc`, the TypeScript compiler.
2. Bundle ES modules into a single umd module using webpack and babel.

---

## Why

Providing both ES modules and a umd package gives developers options. Those working in modern ES2015+ environments can use the ES modules while the umd package supports those working in commonjs or web browser environments. Plus, modern build tools, such as Webpack and rollup, take advantage of ES modules. Build tools will parse, traverse and eliminate unused (dead) code of ES module dependencies.

---

## Why Webpack

Unfortunately, the [TypeScript compiler does not support global fallbacks when compiling to umd](https://github.com/Microsoft/TypeScript/issues/8436). In short, umd packages produced by TypeScript do not support web browsers. Webpack, on the other hand, builds umd packages that support web browsers along with commonjs and amd environments.

---

## Why Babel

To transpile ES2015 to the more widely supported ES5.

---

## Is Compilation Necessary

What about writing TypeScript libraries for TypeScript developers? Is a build process necessary?

Yes. TypeScript does not compile dependencies (packages listed in node_modules). TypeScript libraries need to be compiled to JavaScript before publishing, even if the intended audience is TypeScript developers.

---

## Project Structure

```
Project/
|-- src/
    |-- sayHello.ts
    |-- index.ts
|-- tsconfig.json
|-- webpack.config.js
|-- package.json
```

### sayHello.ts

A simple function to say hello! This is the only functionality of my awesome library.

```js
export function sayHello(name: string): void {
  console.log(`Hello, ${name}`);
}
```

### index.ts

This is the main entry point to the library and, as such, exposes the public API.

```js
export { sayHello } from "./sayHello";
```

### tsconfig.json

The following options instruct the TypeScript compiler to output ES2015 modules.

```json
{
  "compilerOptions": {
    /* Basic Options */
    "target": "ES2015" /* Build to ES2015 */,
    "module": "ES2015" /* using ES2015 modules */,
    "lib": ["es2015", "dom"] /* Using ES2015 features and DOM APIs  */,
    "declaration": true /* Generates corresponding'.d.ts' files. */,
    "declarationDir": "./dist/typings/" /* build '.d.ts' files to ./dist/typeings */,
    "outDir": "./dist/esm/" /* build to ./dist/esm/ */
  },
  "files": ["./src/index.ts"],
  "include": ["./src/**/*.ts"]
}
```

- **declarations**: set to true, `tsc` outputs type definition files which, in turn, provides code intellisense support to TypeScript developers using the library.
- **files** specifies which files to compile. In this case, the main entry point to the library is listed. Compiling the main entry point, index.ts, compiles the rest of the library through dependencies.

### webpack.config.js

Webpack bundles ES modules outputted by `tsc` as a single umd package, targeting ES5. Developers can load the umd package within web browsers, via a script tag, or within commonjs or amd environments.

```js
const path = require("path");

module.exports = (env, argv) => {
  return {
    entry: {
      index: path.resolve(__dirname, "./dist/esm/index.js")
    },
    output: {
      path: path.resolve(__dirname, "./dist/umd"), // builds to ./dist/umd/
      filename: "[name].js", // index.js
      library: "myLibrary", // aka window.myLibrary
      libraryTarget: "umd", // supports commonjs, amd and web browsers
      globalObject: "this"
    },
    module: {
      rules: [{ test: /\.t|js$/, use: "babel-loader" }]
    }
  };
};
```

- **entry** specifies the entry point to the library. In this setup, webpack transpiles ES2015 to ES5 and does not compile TypeScript source code. For this reason, the file outputted by `tsc`, which is an ES2015 module, is specified as the input for webpack.
- **library** specifies the global variable name to use within web browsers.
- **globalObject** sets the global fallback object. The default is `window`, which is not defined in commonjs environments. Setting the global fallback object to `this` generates a umd package that supports both commonjs and web browser environments.
- **module.rules**: In this setup, webpack uses babel to transpile ES2015 code to the more widely supported ES5 syntax.

### package.json

Add the following to package.json

```json
{
  "main": "./dist/umd/index.js",
  "module": "./dist/esm/index.js",
  "types": "./dist/typings/index.d.ts",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack",
    "prebuild": "tsc"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.4",
    "typescript": "^2.7.2",
    "webpack": "^4.1.1",
    "webpack-cli": "^2.0.11"
  }
}
```

- **main** specifies the location of the umd package. Commonjs environments use this as the main entry point to the library.
- **module** specifies the location of the main ES2015 module. Consuming packages that use webpack or rollup will use this as the main entry point to the library and will eliminate unused (dead) code when bundling the library.
- **types** specifies the location of type definition files. Type definition files provide code intellisense and autocompletion support to TypeScript developers using the library.
- **scripts.build** runs `webpack`, transpiling ES2015 modules to an ES5 umd package.
- **scripts.prebuild** runs `tsc`, compiling TypeScript to ES2015 modules. This command automatically runs prior to `npm run build`.
- **devDependencies** lists the minimal set of packages needed to build both ES2015 modules and a umd package. Don't forget to run `npm install`.

---

## Build

```shell
$ npm run build
```

Outputs

![npm-build](/media/npm-build.png)

---

## Consume the built code

### Web Browsers via script tags

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <script src="./path/to/dist/umd/index.js"></script>
    <script>
      // webpack exposes the library as a global, myLibrary
      myLibrary.sayHello("World");
    </script>
  </head>
  <body></body>
</html>
```

### commonjs (Node)

```js
var myLibrary = require("myLibrary");

myLibrary.sayHello("World");
```

### ES modules

```js
import { sayHello } from "myLibrary/esm/sayHello";

sayHello("World");
```

Developers using webpack do not have to specify ES module paths. Webpack will automatically use the ES modules based on the module path specified in package.json.

**webpack**

```js
import { sayHello } from "myLibrary";

sayHello("World");
```
