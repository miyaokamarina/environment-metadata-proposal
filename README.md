# Custom metadata for Lexical Environments and execution context stacks

-   [Protospec][protospec]
-   [Examples / use cases][examples]:
    -   [Reliable React hooks][hooks]
    -   [“Standard” way to define JSX pragma][pragma]
    -   [Zones][zones]
    -   [Informative logs][logs]
    -   [Eliminating closures][closures]

## Open questions

### The name

**Environment Metadata**: LGTM, but probably there are better alternatives.

### How to work with async callbacks

It’s pretty straightforward to pass metadata between contexts when we work with
sync calls. But it becomes more complicated, when we should pass across async
operations (e.g. from the context in which the DOM event handler was set up to
the context in which the callback was called; or the same for NodeJS callbacks).

The Zones protospec states that some of this semantics are
implementation-dependent, but probably there is a better way.

## Semantics explained

-   `Lex` is Lexical Environment.
-   `Rec` is `Lex`’s Environment Record.

`a.js`:

```javascript
// The NewModuleEnvironment abstract operation sets
// Rec.[[Metadata]] to { [[Prototype]]: null }.

import { b } from './b.js';

// rec.[[Metadata]].answer = 42
EnvironmentMetadata.set('answer', 42);

// The PrepareForOrdinaryCall sets b’s _calleeContext_’s
// [[Metadata]] to { [[Prototype]]: Rec.[[Metadata]] }.
b(); // 42
```

`b.js`:

```javascript
// The NewModuleEnvironment abstract operation sets
// Rec.[[Metadata]] to { [[Prototype]]: null }.

export const b = () => {
    // Rec.[[Metadata]].answer
    return EnvironmentMetadata.get('answer');
};
```

[protospec]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/PROTOSPEC.md
[examples]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md
[hooks]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md#reliable-react-hooks
[pragma]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md#standard-way-to-define-jsx-pragma
[zones]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md#zones
[logs]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md#informative-logs
[closures]: https://github.com/miyaokamarina/environment-metadata-proposal/blob/master/EXAMPLES.md#eliminating-closures
