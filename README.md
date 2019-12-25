# Custom metadata for Lexical Environments

## Quick example

```typescript
declare const LexicalMetadata: {
    readonly set: (key: unknown, value?: unknown) => void;
    readonly has: (key: unknown) => boolean;
    readonly get: (key: unknown) => unknown;
    readonly delete: (key: unknown) => void;
};

// env0 = NewModuleEnvironment(...)
LexicalMetadata.set('x', 'a');

function f() {
    // callSiteEnv implcitly received from call site
    // env1 = NewFunctionEnvironment(f, undefined)
    // env1.[[Parent]] = env0
    // env1.[[CallSiteEnv]] = callSiteEnv // env2

    return LexicalMetadata.get('x');
    // 1. Lookup in `env1`’s `[[CallSiteEnv]]` chain.
    // 2. If no metadata for the `'x'` key found in this chain, Then
    // 3.     Lookup in `env1`’s `[[Parent]]` chain.
    // 4. If still no metadata for the `'x'` key found, Then
    // 5. return `undefined`.
}

function g() {
    // callSiteEnv implcitly received from call site
    // env2 = NewFunctionEnvironment(g, undefined)
    // env2.[[Parent]] = env0
    // env2.[[CallSiteEnv]] = callSiteEnv // env3

    // `env2` implicitly passed to `f` as callSiteEnv
    f();
}

function h() {
    // callSiteEnv implcitly received from call site
    // env3 = NewFunctionEnvironment(h, undefined)
    // env3.[[Parent]] = env0
    // env3.[[CallSiteEnv]] = callSiteEnv // env0
    LexicalMetadata.set('x', 'b');

    // `env3` implicitly passed to `g` as callSiteEnv
    return g(); // should return `'b'`
}

// `env0` implicitly passed to `h` as callSiteEnv
console.log(h());
```

## Use cases

### Reliable React hooks

Currently, React hooks rely on weird hacks, which require [unobvious restrictions on them](https://reactjs.org/docs/hooks-rules.html). When using this API, hooks may be implemented in more reliable way, allowing _most_ hooks to be called from within conditionals, loops, callbacks, etc.

### “Standard” way to define JSX pragma

The following code…

```jsx
// `Symbol.jsx{Factory,Fragment}` polyfill ↓
import 'symbol-jsx';

// Custom JSX factory and fragment symbols ↓
import { h, Fragment } from '@foo/bar';

LexicalMetadata.set(Symbol.jsxFactory, h);
LexicalMetadata.set(Symbol.jsxFragment, Fragment);

const a = <a href='https://example.com/'>Example</a>;
```

… transpiles to…

```javascript
import 'symbol-jsx';
import { h, Fragment } from '@foo/bar';
import { __createElement } from '@babel/runtime';

LexicalMetadata.set(Symbol.jsxFactory, h);
LexicalMetadata.set(Symbol.jsxFragment, Fragment);

const a = __createElement('a', { href: 'https://example.com/' }, 'Example');
```

### Zones (future)

The functionality of withdrawn Zones proposal may be implemented using this API.

### Eliminating closures (future)

## Relations with other proposals

## New Other Properties of the Global Object

### `LexicalMetadata`

## New Abstract Methods of the Environment Record

### `SetLexicalMetadata(key, value)`

### `HasLexicalMetadata(key)`

### `GetLexicalMetadata(key)`

### `DeleteLexicalMetadata(key)`

## Changes to Runtime Semantics

### `EvaluateCall`

### `EvaluateNew`
