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

```jsx
const symbolContext = Symbol();

const createElement = (Component, props, ...children) => {
    ...
    LexicalMetadata.set(symbolContext, context);
    ...
    const result = Component(props);
    ...
};

const useContext = Context => {
    return LexicalMetadata.get(symbolContext)[Context.symbol];
};

const Component = props => {
    return <div>{array.map(x => {
        useContext(L10n).translate(x);
    })}</div>
};
```

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

Unlike current approach (using compiler options or the `@jsx` comment), lexical metadata approach allows

1. to define JSX factory and fragment in compiler-independed fashion. I.e., Babel, TypeScript or any other compiler may use pragmas defined in a lexical environment in semi-standard way,
2. to redefine JSX factory and fragment in nested lexical environments. E.g., in edge cases, when usage of multiple JSX libraries in one file is desired.

The `__createElement` helper may be defined in helpers library, added to transpiled file, or even defined in a lirary like `symbol-observable`.

### Zones (future)

The functionality of withdrawn Zones proposal may be implemented using this API.

**TODO:** Provide minimal example.

### Eliminating closures (future)

**TODO:** Provide minimal example.

## New Other Properties of the Global Object

### `LexicalMetadata`

## New Abstract Methods of the Environment Record

### `SetLexicalMetadata(key, value)`

### `HasLexicalMetadata(key)`

### `GetLexicalMetadata(key)`

### `DeleteLexicalMetadata(key)`

## Changes to Runtime Semantics

### 7.3.12 Call ( F, V, env \[ , argumentsList ] )

### 12.3.6.2 EvaluateCall ( func, ref, arguments, tailPosition, env )

The abstract operation EvaluateCall takes as arguments a value _func_, a value _ref_, a Parse Node _arguments_, a Boolean argument _tailPosition_, and an Environment Record _env_. It performs the following steps:

1.  If Type(_ref_) is Reference, then
    1. If IsPropertyReference(_ref_) is **true**, then
        1. Let _thisValue_ be GetThisValue(_ref_).
    2. Else,
        1. Assert: the base of _ref_ is an Environment Record.
        2. Let _refEnv_ be GetBase(_ref_).
        3. Let _thisValue_ be _refEnv_.WithBaseObject().
2.  Else,
        1. Let _thisValue_ be **undefined**.
3.  Let _argList_ be ? ArgumentListEvaluation of arguments.
4.  If Type(_func_) is not Object, throw a **TypeError** exception.
5.  If IsCallable(_func_) is **false**, throw a **TypeError** exception.
6.  If _tailPosition_ is **true**, perform PrepareForTailCall().
7.  Let _result_ be Call(_func_, _thisValue_, _argList_).
8.  Assert: If _tailPosition_ is **true**, the above call will not return here, but instead evaluation will continue as if the following return has already occurred.
9.  Assert: If _result_ is not an abrupt completion, then Type(_result_) is an ECMAScript language type.
10. Return _result_.

### EvaluateNew
