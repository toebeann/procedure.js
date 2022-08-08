# procedure.js
The simple RPC framework for Node.js.

<!-- [![NPM version](https://img.shields.io/npm/v/@toebeann/procedure.js.svg)](https://npmjs.org/package/@toebeann/procedure.js "View procedure.js on NPM") [![NPM downloads](https://img.shields.io/npm/dw/@toebeann/procedure.js.svg)](https://npmjs.org/package/@toebeann/procedure.js "View procedure.js on NPM") -->
<!-- [![Code quality](https://img.shields.io/codefactor/grade/github/toebeann/procedure.js.svg)](https://www.codefactor.io/repository/github/toebeann/procedure.js "Check code quality on CodeFactor") -->
<!-- [![Coverity Scan build status](https://img.shields.io/coverity/scan/.svg)](https://scan.coverity.com/projects/toebeann-procedure.js "View build status on Coverity Scan") -->
<!-- ![nyc code coverage](https://img.shields.io/nycrc/toebeann/procedure.js.svg)] -->
[![Twitter Follow](https://img.shields.io/twitter/follow/toebean__.svg?style=social)](https://twitter.com/toebean__ "Follow @toebean__ on Twitter") <!-- [![GitHub Sponsors donation button](https://img.shields.io/badge/github-donate-yellow.svg)](https://github.com/sponsors/toebeann "Donate to procedure.js with GitHub Sponsors") --> [![PayPal donation button](https://img.shields.io/badge/paypal-donate-yellow.svg)](https://paypal.me/tobeyblaber "Donate to procedure.js with PayPal")


## Description
A lightweight alternative to the boilerplate-heavy gRPC, or spinning up a HTTP server through which to run a REST API. Procedure bridges the gap between independent applications and services with as little code as possible, so that you can just focus on building your app!

```js
// my-app/index.js

// a simple procedure which returns the square of a given number
const procedure = new Procedure('tcp://*:5000', (n) => n ** 2).bind();
```

```js
// some-other-app/index.js

// calling the procedure to find the square of 8
let squared = await Procedure.call('tcp://localhost:5000', 8);
console.log(squared); //outputs 64
```

Procedure allows you to define procedures which can be called over [TCP](#tcp-intrainter-network-over-tcpip), [WebSockets](#ws-intrainter-network-over-websockets), [IPC](#ipc-interprocess), and [across threads or modules in the same process](#inproc-intraprocess). Use whichever [transport](#transports-more-than-just-tcp) is most appropriate for your use case, or mix-and-match!

<!-- Placeholder for when the .NET implementation is ready

With [implementations in multiple languages](#language-implementations), applications written in C#, JavaScript, and others can interface together with ease!

-->

## Table of contents
- [procedure.js](#procedurejs)
  - [Description](#description)
  - [Table of contents](#table-of-contents)
  - [Usage](#usage)
    - [`async`/`await`](#asyncawait)
    - [Parameters and return types](#parameters-and-return-types)
      - [A note about `null` and `undefined`](#a-note-about-null-and-undefined)
        - [Optional parameter support](#optional-parameter-support)
        - [`null` and `undefined` properties](#null-and-undefined-properties)
      - [Pass by reference?](#pass-by-reference)
    - [Error handling](#error-handling)
  - [Transports: More than just TCP!](#transports-more-than-just-tcp)
    - [INPROC: intraprocess](#inproc-intraprocess)
    - [IPC: intra/interprocess](#ipc-intrainterprocess)
    - [TCP: intra/inter-network over TCP/IP](#tcp-intrainter-network-over-tcpip)
    - [WS: intra/inter-network over WebSockets](#ws-intrainter-network-over-websockets)
  - [Handling breaking changes to your procedures](#handling-breaking-changes-to-your-procedures)
  - [Language implementations](#language-implementations)
  - [License](#license)

<!-- ## Install

### [npm](https://npmjs.com "npm is a package manager for JavaScript")
- Install : `npm install --save @toebeann/procedure.js`
- Import: `import Procedure from '@toebeann/procedure.js'`
- Require: `const Procedure = require('@toebeann/procedure.js')` -->

## Usage
With Procedure, setting up your function to be called from another process (whether remote or local) is remarkably simple:
```js
const procedure = new Procedure('tcp://*:5000', (n) => n ** 2);
procedure.bind();
```

And calling it is just as easy:
```js
let x = 8;
let xSquared = await Procedure.call('tcp://localhost:5000', x);
console.log(xSquared); //outputs 64
console.log(typeof xSquared); // outputs 'number'
```

### `async`/`await`
Asynchronous functions are fully supported:
```js
const procedure = new Procedure('tcp://127.0.0.1:8888', async () => {
    const response = await fetch('https://catfact.ninja/fact');
    if (response.ok) {
        return (await response.json()).fact;
    } else {
        throw new Error(`${response.status}: ${response.statusText}`);
    }
});
```

### Parameters and return types
Parameter and return types can be anything supported by the [msgpack](https://github.com/msgpack/msgpack-javascript) serialization format, which covers much of JavaScript by default, and you can handle unsupported types with [Extension Types](https://github.com/msgpack/msgpack-javascript#extension-types). We generally recommend sticking to [PODs](https://en.wikipedia.org/wiki/Passive_data_structure "plain old data objects"). It is possible to pass more complex types around in many cases - but note that they will be passed by value, [not by reference](#pass-by-reference).

Procedure supports a single parameter, or none. We considered supporting multiple parameters, but this increases the complexity of the design and leads to potentially inconsistent APIs for different language implementations, while multiple parameters can easily be simulated through the use of [PODs](https://en.wikipedia.org/wiki/Passive_data_structure "plain old data objects") (e.g. object literals, property bags) or arrays in virtually any programming language.

If you have existing functions with multiple parameters which you want to expose as procedures, wrapping them is trivial:
```js
function myFunction(a, b, c) {
    return a + b * c;
}

const procedure = new Procedure('tcp://*:30666', params => myFunction(...params)).bind();
```
Which can then be called like so:
```js
Procedure.call('tcp://localhost:30666', [1, 2, 3]);
```
For functions where you have optional parameters, it might make more sense to use object literals/property bags instead of arrays.

Functions which accept multiple parameters where only the first is required (or none) will work as is, but you will only be able to pass the first parameter via `Procedure.call`.

#### A note about `null` and `undefined`
##### Optional parameter support
In the JavaScript implementation of msgpack, [`undefined` is mapped to `null`](https://github.com/msgpack/msgpack-javascript#messagepack-mapping-table). This means that all `undefined` values will be decoded as `null`, and there is no way to differentiate between the two.

This causes an issue for procedures which accept an optional parameter, as in most implementations of optional parameters in JavaScript, only `undefined` is coerced into a default value.

It also means that procedures with no return value will evaluate to `null` instead of `undefined`, which could cause unexpected behavior if you were to pass the return value of a procedure into another function as an optional parameter.

To handle these inconsistencies, we coerce a msgpack decoded `null` to `undefined`. This does not affect the properties of objects - they will still be evaluated as `null` when they were either `null` or `undefined`.

To disable this behavior, you can set `optionalParameterSupport` to `false` for either procedure definitions or calls, or both:
```js
const procedure = new Procedure(
    'tcp://*:54321',
    x => { ... },
    { optionalParameterSupport: false }
);
```

```js
await Procedure.call('tcp://*:54321', x, { optionalParameterSupport: false });
```
Note that disabling at the definition will not affect the return value, and disabling at the call will not affect the input parameter.

##### `null` and `undefined` properties
For objects, we do not coerce `null` properties to `undefined`. Instead, we leave them as is, but strip any properties with the value of `undefined` from the object prior to transmission, thereby allowing those properties to be evaluated as `undefined` at the other end, while `null` properties remain `null`.

This operation adds some overhead, and any code that relies on the presence of a property to infer meaning may not work as expected, e.g. `if ('prop' in obj)`.

To disable this behavior, you can set the `stripUndefinedProperties` option to `false` for either procedure definitions or calls, or both:
```js
const procedure = new Procedure(
    'tcp://*:54321',
    x => {
        ...
    },
    { stripUndefinedProperties: false }
);
```

```js
await Procedure.call('tcp://*:54321', x, { stripUndefinedProperties: false });
```
Note that disabling at the definition will not affect the return value, and disabling at the call will not affect the input parameter.

#### Pass by reference?
It is **impossible** to pass by reference with Procedure. All data is encoded and then sent across the wire, similar to what happens when a REST API responds to a request by sending back a JSON string/file. You can parse that JSON into an object and access its data, but you only have a *copy* of the data that lives on the server.

For example, if you were to implement the following procedure:
```js
const procedure = new Procedure('tcp://*:33333', x => x.foo = "bar");
```
And then call it like so:
```js
let obj = { foo: 123 };
await Procedure.call('tcp://*:33333', obj);
console.log(obj); // outputs '{ foo: 123 }'
```
The `obj` object would remain unchanged, because the procedure is acting on a *clone* of the object, not the object itself. First, the object is encoded for transmission by msgpack, then sent across the wire by nanomsg, and finally decoded by msgpack at the other end into a brand new object.

### Error handling
Errors are thrown back to the caller, enabling you to react to and handle them the way you would as if the functions were defined locally:
```js
let x = { foo: 'bar' };
let xSquared = await Procedure.call('tcp://localhost:5000', x);
// throws `SyntaxError: expected expression, got '**'
```

## Transports: More than just TCP!
The examples in this readme all use TCP to demonstrate the most common use case for RPC. However, Procedure is built on top of [nanomsg](https://nanomsg.org/), which means it supports all of the same transports that nanomsg does:

### INPROC: intraprocess
Call functions between threads or modules of the same process.
- `inproc://foobar`

### IPC: intra/interprocess
Call functions between different processes on the same host.
- `ipc://foobar.ipc`
- `ipc:///tmp/test.ipc`
- `ipc://MyApp/MyProcedure`

### TCP: intra/inter-network over TCP/IP
Call functions between processes across TCP with support for both IPv4 addresses and DNS names*. IPv6 support coming soon!
- `tcp://*:80`
- `tcp://192.168.0.5:5600`
- `tcp://localhost:33000`*

TLS (`tcp+tls://`) is not currently supported.

_<sub>* DNS names are only supported when calling a procedure, not when defining.</sub>_

### WS: intra/inter-network over WebSockets
Call functions between processes across WebSockets over TCP with support for both IPv4 address and DNS names*. IPv6 support coming soon!
- `ws://*`
- `ws://127.0.0.1:8080`
- `ws://example.com`*

TLS (`wss://`) is not currently supported.

_<sub>* DNS names are only supported when calling a procedure, not when defining.</sub>_

## Handling breaking changes to your procedures
Procedure has no way of knowing what the parameter or return types of the procedure at the other end of the call will be. If you rewrite a procedure to return a different type or to accept a different parameter type, you will only get errors at runtime, not at compile time.

Therefore, if you are developing procedures for public consumption, be mindful of the fact that **breaking changes on the same endpoint will result in unhappy consumers!**

If you do need to make breaking changes to a procedure, it is recommended to either:
- implement the breaking changes on a new endpoint, while keeping the original available:
  ```js
  myFunction(x) {
      return isNaN(x); // return boolean indicating whether x is NaN
  }

  myFunctionV2(x) {
      if (isNaN(x)) {
          ...
          // do stuff with x when it is NaN
          ...
          return true;
      }
      // breaking change, we no longer return a boolean in all cases
  }

  const procedure = new Procedure('tcp://*:33000', myFunction).bind();
  const procedureV2 = new Procedure('tcp://*:33001', myFunctionV2).bind();
  ```

  ```js
  const v1Result = await Procedure.call('tcp://localhost:33000'); // returns false
  const v2Result = await Procedure.call('tcp://localhost:33001'); // returns undefined
  ```
- use a parameter or property to specify a version modifier, defaulting to the original when unspecified:
  ```js
  myFunction(x) {
      return isNaN(x); // return boolean indicating whether x is NaN
  }

  myFunctionV2(x) {
      if (isNaN(x)) {
          ...
          // do stuff with x when it is NaN
          ...
          return true;
      }
      // breaking change, we no longer return a boolean in all cases
  }

  const procedure = new Procedure('tcp://*:33000', options => {
      switch (options?.version) {
          case 2: return myFunctionV2(options.x);
          default: return myFunction(options.x);
      }
  });
  procedure.bind();
  ```

  ```js
  const v1Result = await Procedure.call('tcp://localhost:33000'); // returns false
  const v2Result = await Procedure.call('tcp://localhost:33000', { version: 2 }); //returns undefined
  ```

  You may prefer to use a [semver](https://www.npmjs.com/package/semver) compatible string for versioning.

<!-- TODO: API reference

## API reference

-->

## Language implementations
As Procedure is designed around nanomsg and msgpack, it can be implemented in any language that has both a nanomsg binding and a msgpack implementation.

Presently, the only official implementation of Procedure is procedure.js for Node.js, but a .NET implementation written in C# and a stripped-down browser library for calling procedures via the [WS transport](#ws-intrainter-network-over-websockets) are currently being worked on.

<!-- Placeholder for when the .NET and/or browser implementation is ready

The different language implementations of Procedure are fully compatible with one another. So, for example, a procedure written in a JavaScript Node.js application can be called by a .NET application written in C#, and vice versa!

Procedure is currently implemented in the following languages:
<table>
    <thead>
        <tr>
            <th>Implementation</th>
            <th colspan=2>Language</th>
            <th>Notes</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan=3>
                <a href="https://github.com/toebeann/Procedure.NET" target="_blank">Procedure.NET</a>
            </td>
            <td>C#</td>
            <td rowspan=3>.NET</td>
            <td rowspan=3>
                Implemented in C#, not yet confirmed to work with F# and VB - YMMV!
            </td>
        </tr>
        <tr>
            <td>F#</td>
        </tr>
        <tr>
            <td>Visual Basic</td>
        </tr>
        <tr>
            <td rowspan=2>
                <a href="https://github.com/toebeann/procedure.js" target="_blank">procedure.js</a>
            </td>
            <td>JavaScript</td>
            <td rowspan=2>Node.js</td>
            <td rowspan=2>
                For Node.js apps only. For a library designed for browser use, see <a href="https://github.com/toebeann/procedure.ws" target="_blank">procedure.ws</a>
            </td>
        </tr>
        <tr>
            <td>TypeScript</td>
        </tr>
        <tr>
            <td rowspan=2>
                <a href="https://github.com/toebeann/procedure.ws" target="_blank">procedure.ws</a>
            </td>
            <td>JavaScript</td>
            <td rowspan=2>Browser</td>
            <td rowspan=2>
                Designed for use in the browser, only calling procedures via the <a href="#ws-intrainter-network-over-websockets">WS transport</a> is available. For a fully featured Node.js implementation, see <a href="https://github.com/toebeann/procedure.js" target="_blank">procedure.js</a>
            </td>
        </tr>
        <tr>
            <td>TypeScript</td>
        </tr>
    </tbody>
</table>

-->

If you would like to contribute a Procedure implementation in another language, please feel free! Create a GitHub repository for the language implementation and open an issue with us once it's ready for review! 💜

## License
procedure.js is licensed under [MIT](LICENSE).
