# Custom metadata for Lexical Environments and execution context stacks

## Minimal example

`a.js`:

```javascript
// let rec = LexicalEnvironment.EnvironmentRecord
// rec.[[CalleeMetadata]] here is { [[Prototype]]: null }

import { b } from './b.js';

LexicalMetadata.set('answer', 42);
// rec.[[CalleeMetadata]].set('answer', 42)

b(); // 42
// b’s [[CallerMetadata]] is set to {
//     ...rec.[[CalleeMetadata]],
//     [[Parent]]: rec.[[CallerMetadata]], // `null`
// } in PrepareForOrdinaryCall.
```

`b.js`:

```javascript
// let rec = LexicalEnvironment.EnvironmentRecord
// rec.[[CalleeMetadata]] here is { [[Prototype]]: null }

export const b = () => {
    // let rec = LecicalEnvironment.EnvironmentRecord
    // rec.[[CalleeMetadata]] here is { [[Prototype]]: { [[Prototype]]: null } }
    // rec.[[CallerMetadata]] here is { 'answer': 42, [[Prototype]]: null }

    return LexicalMetadata.get('answer');
    // 'answer' in rec.[[CallerMetadata]] ? rec.[[CallerMetadata]].answer : rec.[[CalleeMetadata]].answer
};
```

## Use cases

### Reliable React hooks

Currently, React hooks rely on weird hacks, which require
[unobvious restrictions on them](https://reactjs.org/docs/hooks-rules.html).
When using this API, hooks may be implemented in more reliable way, allowing
_most_ hooks to be called from within conditionals, loops, callbacks, etc.

```jsx
const symbolContext = Symbol();

const createElement = (Component, props, ...children) => {
    // ...
    LexicalMetadata.set(symbolContext, context);
    // ...
    const result = Component(props);
    // ...
};

const useContext = Context => {
    return LexicalMetadata.get(symbolContext)[Context.symbol];
};

const Component = props => {
    return (
        <div>
            {array.map(x => {
                useContext(L10n).translate(x);
            })}
        </div>
    );
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

Unlike current approach (using compiler options or the `@jsx` comment), lexical
metadata approach allows

1. to define JSX factory and fragment in compiler-independed fashion. I.e.,
   Babel, TypeScript or any other compiler may use pragmas defined in a lexical
   environment in semi-standard way,
2. to redefine JSX factory and fragment in nested lexical environments. E.g., in
   edge cases, when usage of multiple JSX libraries in one file is desired.

The `__createElement` helper may be defined in helpers library, added to
transpiled file, or even defined in a lirary like `symbol-observable`.

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

### 9.2.1.1 PrepareForOrdinaryCall ( _F_, _newTarget_ )

1.  Assert: Type(_newTarget_) is Undefined or Object.
2.  Let _callerContext_ be the running execution context.
3.  Let _calleeContext_ be a new ECMAScript code execution context.
4.  Set the Function of _calleeContext_ to _F_.
5.  Let _calleeRealm_ be _F_.\[\[Realm]].
6.  Set the Realm of _calleeContext_ to _calleeRealm_.
7.  Set the ScriptOrModule of _calleeContext_ to _F_.\[\[ScriptOrModule]].
8.  Let _localEnv_ be NewFunctionEnvironment(_F_, _newTarget_).
9.  Perform PrepareCelleeEnvironment(_callerContext_, _calleeContext_).
10. Set the LexicalEnvironment of _calleeContext_ to _localEnv_.
11. Set the VariableEnvironment of _calleeContext_ to _localEnv_.
12. If _callerContext_ is not already suspended, suspend _callerContext_.
13. Push _calleeContext_ onto the execution context stack; _calleeContext_ is now the running execution context.
14. NOTE: Any exception objects produced after this point are associated with _calleeRealm_.
15. Return _calleeContext_.

### ?? PrepareCelleeEnvironment ( _callerContext_, _calleeContext_ )

When the PrepareCalleeEnvironment abstract operation is called with execution context _callerContext_ and execution context _calleeContext_, the following steps are taken:

1. Let _callerRec_ be EnvironmentRecord of _callerContext_'s LexicalEnvironment.
2. Let _calleeRec_ be EnvironmentRecord of _calleeContext_'s LexicalEnvironment.
3. Set _calleeRec_.\[\[CallerMetadata]] to ! ObjectCreate(_callerRec_.\[\[CalleeMetadata]]).
4. Return NormalCompletion(`empty`).
















