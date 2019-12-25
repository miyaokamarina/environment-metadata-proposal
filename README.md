# Custom metadata for Lexical Environments

## Quick example

```typescript
declare const LexicalMetadata: {
    readonly set: (key: unknown, value?: unknown) => void;
    readonly has: (key: unknown) => boolean;
    readonly get: (key: unknown) => unknown;
    readonly delete: (key: unknown) => void;
};

// env0 = new ModuleEnvironment
LexicalMetadata.set('x', 'a');

function f() {
    // callSiteEnv implcitly received from call site
    // env1 = new FunctionEnvironment
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
    // env2 = new FunctionEnvironment
    // env2.[[Parent]] = env0
    // env2.[[CallSiteEnv]] = callSiteEnv // env3

    // `env2` implicitly passed to `f` as callSiteEnv
    f();
}

function h() {
    // callSiteEnv implcitly received from call site
    // env3 = new FunctionEnvironment
    // env3.[[Parent]] = env0
    // env3.[[CallSiteEnv]] = callSiteEnv // env0
    LexicalMetadata.set('x', 'b');

    // `env3` implicitly passed to `g` as callSiteEnv
    return g(); // should return `'b'`
}

// `env0` implicitly passed to `h` as callSiteEnv
console.log(h());
```

## Case studies

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
