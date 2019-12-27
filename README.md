# Custom metadata for Lexical Environments and execution context stacks

## Open questions

### The name

**Environment Metadata** LGTM, but probably there are better alternatives.

### API

-   Currently, the `EnvironmentMetadata` is a regular object with method to
    access environment metadata properties. Maybe, it should be an exotic object
    with custom \[\[Get]]/\[\[Set]]/etc. internal slots?
-   Maybe, it should allow more granular and low-level operations on metadata?
    E.g., explicit access to the \[\[CalleeMetadata\]\] (lexical),
    \[\[CallerMetadata]] (passed from caller to callee through stack),
    \[\[ReturnMetadata\]\] (passed from callee to caller through stack).

### How to work with async callbacks

It’s pretty straightforward to pass metadata between contexts, when we work with
sync calls. But it becomes more complicated, when we should pass across async
operations (e.g. from the context in which the DOM event handler was set up to
the context in which the callback was called; or the same for NodeJS callbacks).

The Zones protospec states that some of this semantics are
implementation-dependent, but probably there is a better way.

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
    EnvironmentMetadata.set(symbolContext, context);
    // ...
    const result = Component(props);
    // ...
};

const useContext = Context => {
    return EnvironmentMetadata.get(symbolContext)[Context.symbol];
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

> **NOTE:** With some changes to the Hooks API (`useState<s>(initial: s)` →
> `useState<s>(name: PropertyKey, initial: s)`), even the `useState` hook may be
> called from everywhere within a component — or even from outside of component.

### “Standard” way to define JSX pragma

As number of JSX frameworks grows, it is necessary to fine-tune compiler to use
various pragmas — in some cases you may need to use two different pragmas in one
file (e.g., if you transitiononing from Vue to React or vice versa, and use
mixed React+Vue components).

Using this API, it is possible to intoroduce conventinal framework-agnostic
symbols and pragmas, which will be used by frameworks.

The following code…

```jsx
import { symbolFactory, symbolFragment } from 'universal-jsx';

// Custom JSX factory and fragment symbols ↓
import { h, Fragment } from '@foo/bar';

EnvironmentMetadata.set(symbolFactory, h);
EnvironmentMetadata.set(symbolFragment, Fragment);

const x = (
    <>
        Hello, world<em>‼</em>
    </>
);
const y = <a href='https://example.com/'>{x}</a>;
```

… transpiles to…

```javascript
import { symbolFactory, symbolFragment, $h, $Fragment } from 'universal-jsx';
import { h, Fragment } from '@foo/bar';

EnvironmentMetadata.set(symbolFactory, h);
EnvironmentMetadata.set(symbolFragment, Fragment);

const x = $h($Fragment, null, 'Hello, world', $h('em', null, '‼'));
const y = $h('a', { href: 'https://example.com/' }, x);
```

Unlike current approach (using compiler options or the `@jsx` comment),
environment metadata approach allows

1. to define JSX factory and fragment in compiler-independed fashion. I.e.,
   Babel, TypeScript or any other compiler may use pragmas defined in a lexical
   environment in semi-standard way,
2. to redefine JSX factory and fragment in nested lexical environments. E.g., in
   edge cases, when usage of multiple JSX libraries in one file is desired.

> **NOTE:** What if I want to write the `setupJsx(factory, fragment)` helper and
> use it instead of direct `EnvironmentMetadata.*` calls? Currently, it cannot
> change callee environment, we need a way to propagate **some** environment
> changes from callee back to caller.

### Zones

Some functionality of withdrawn Zones proposal may be implemented using this
API.

**TODO:** Provide minimal example.

### Informative logs

**TODO:** Provide minimal example.

### Eliminating closures

Closures\* may be unwanted in some cases due to its overheads. This API allows
to pass data to functions indirectly and without closures.

\* Closures defined in places other than script/module top level.

Note that all functions defined at the top level, so no expensive closures are
created neither on component render nor inside loop.

```jsx
const onClickSymbol = Symbol();
const itemSymbol = Symbol();

const handleClick = () => {
    const item = EnvironmentMetadata.get(itemSymbol);
    const onClick = EnvironmentMetadata.get(onClickSymbol);

    onClick(item);
};

const map = item => {
    EnvironmentMetadata.set(itemSymbol, item);

    return (
        <div key={item} onClick={handleClick}>
            {item}
        </div>
    );
};

const List = ({ items, onClick }) => {
    EnvironmentMetadata.set(onClickSymbol, onClick);

    return <div>{items.map(items)}</div>;
};

const onClick = console.log;

const list = <List items={['a', 'b']} onClick={onClick} />;

// On click on `a` list item, `'a'` string will be printed to the console.
```

### Semantics explained

`a.js`:

```javascript
// let rec = LexicalEnvironment.EnvironmentRecord
//
// The NewModuleEnvironment abstract operation sets
// rec.[[CalleeMetadata]] to {
//     [[Prototype]]: LexicalEnvironment
//         .outer
//         .EnvironmentRecord
//         .[[CalleeMetadata]]
// }.

import { b } from './b.js';

// rec.[[CalleeMetadata]].answer = 42
EnvironmentMetadata.set('answer', 42);

// The PrepareForOrdinaryCall sets b’s
// _calleeContext_’s [[CallerMetadata]] to {
//     [[Prototype]]: rec.[[CalleeMetadata]]
// }.
b(); // 42
```

`b.js`:

```javascript
// let rec = LexicalEnvironment.EnvironmentRecord
//
// The NewModuleEnvironment abstract operation sets
// rec.[[CalleeMetadata]] to {
//     [[Prototype]]: LexicalEnvironment
//         .outer
//         .EnvironmentRecord
//         .[[CalleeMetadata]]
// }.

export const b = () => {
    // let rec = LexicalEnvironment.EnvironmentRecord
    //
    // The NewFunctionEnvironment abstract operation sets
    // rec.[[CalleeMetadata]] to {
    //     [[Prototype]]: LexicalEnvironment
    //         .outer
    //         .EnvironmentRecord
    //         .[[CalleeMetadata]]
    // }.

    // 'answer' in rec.[[CallerMetadata]]
    //     ? rec.[[CallerMetadata]].answer
    //     : rec.[[CalleeMetadata]].answer
    return EnvironmentMetadata.get('answer');
};
```

## Protospec

### ?? LexicalMetadata

**TODO:** _New other property of Global Object._

### <ins>?? SetEnvironmentMetadata ( _propertyKey_, _value_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. Perform ! _callerMeta_.\[\[Set]](_key_, _value_, _callerMeta_).
6. Perform ! _calleeMeta_.\[\[Set]](_key_, _value_, _calleeMeta_).
7. Return NormalCompletion(`empty`).

> **NOTE:** A property must be set in both caller and callee metadata. Caller
> metadata takes priority when resding properties using HasEnvironmentMetadata
> and GetEnvironmentMetadata, and callee metadata used to inherit from when
> passing to functions called from current context.

### <ins>?? HasEnvironmentMetadata ( _propertyKey_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. If ! _callerMeta_.\[\[HasProperty]](_key_) is **true**, then
    1. Return **true**.
6. Else,
    1. Return ! _calleeMeta_.\[\[HasProperty]](_key_).

### <ins>?? GetEnvironmentMetadata ( _propertyKey_ )</ins>

1. Let _key_ be ? ToPropertyKey(_propertyKey_).
2. Let _envRec_ be EnvironmentRecord of current execution context's
   LexicalEnvironment.
3. Let _callerMeta_ be _envRec_.\[\[CallerMetadata]].
4. Let _calleeMeta_ be _envRec_.\[\[CalleeMetadata]].
5. If ! _callerMeta_.\[\[HasProperty]](_key_) is **true**, then
    1. Return ! _callerMeta_.\[\[Get]](_key_, _callerMeta_).
6. Else,
    1. Return ! _calleeMeta_.\[\[Get]](_key_, _calleeMeta_).

### ?? DeleteEnvironmentMetadata ( _propertyKey_ )

### 9.2.1.1 PrepareForOrdinaryCall ( _F_, _newTarget_ )

1.  Assert: Type(_newTarget_) is Undefined or Object.
2.  Let _callerContext_ be the running execution context.
3.  Let _calleeContext_ be a new ECMAScript code execution context.
4.  Set the Function of _calleeContext_ to _F_.
5.  Let _calleeRealm_ be _F_.\[\[Realm]].
6.  Set the Realm of _calleeContext_ to _calleeRealm_.
7.  Set the ScriptOrModule of _calleeContext_ to _F_.\[\[ScriptOrModule]].
8.  Let _localEnv_ be NewFunctionEnvironment(_F_, _newTarget_).
9.  <ins>Perform PrepareCelleeEnvironment(_callerContext_,
    _calleeContext_).</ins>
10. Set the LexicalEnvironment of _calleeContext_ to _localEnv_.
11. Set the VariableEnvironment of _calleeContext_ to _localEnv_.
12. If _callerContext_ is not already suspended, suspend _callerContext_.
13. Push _calleeContext_ onto the execution context stack; _calleeContext_ is
    now the running execution context.
14. NOTE: Any exception objects produced after this point are associated with
    _calleeRealm_.
15. Return _calleeContext_.

### <ins>?? PrepareCelleeEnvironment ( _callerContext_, _calleeContext_ )</ins>

When the PrepareCalleeEnvironment abstract operation is called with execution
context _callerContext_ and execution context _calleeContext_, the following
steps are taken:

1. Let _callerRec_ be EnvironmentRecord of _callerContext_'s LexicalEnvironment.
2. Let _calleeRec_ be EnvironmentRecord of _calleeContext_'s LexicalEnvironment.
3. Set _calleeRec_.\[\[CallerMetadata]] to !
   ObjectCreate(_callerRec_.\[\[CalleeMetadata]]).
4. Return NormalCompletion(`empty`).

### 8.4.1 EnqueueJob ( _queueName_, _job_, _arguments_ )

1.  Assert: Type(_queueName_) is String and its value is the name of a Job Queue
    recognized by this implementation.
2.  Assert: _job_ is the name of a Job.
3.  Assert: _arguments_ is a List that has the same number of elements as the
    number of parameters required by _job_.
4.  Let _callerContext_ be the running execution context.
5.  Let _callerRealm_ be _callerContext_'s Realm.
6.  Let _callerScriptOrModule_ be _callerContext_'s ScriptOrModule.
7.  <ins>Let _callerRec_ be EnvironmentRecord of _callerContext_'s
    LexicalEnvironment.</ins>
8.  <ins>Let _callerMeta_ be _callerRec_.\[\[CalleeMetadata]].</ins>
9.  Let _pending_ be PendingJob { \[\[Job]]: _job_, \[\[Arguments]]:
    _arguments_, \[\[Realm]]: _callerRealm_, \[\[ScriptOrModule]]:
    _callerScriptOrModule_, \[\[HostDefined]]: **undefined**<ins>,
    \[\[CallerMetadata]]: _callerMeta_</ins> }.
10. Perform any implementation or host environment defined processing of
    _pending_. This may include modifying the \[\[HostDefined]] field or any
    other field of pending.
11. Add _pending_ at the back of the Job Queue named by _queueName_.
12. Return NormalCompletion(`empty`).

### 8.6 RunJobs ( )

1. Perform ? InitializeHostDefinedRealm().
2. In an implementation-dependent manner, obtain the ECMAScript source texts
   (see clause 10) and any associated host-defined values for zero or more
   ECMAScript scripts and/or ECMAScript modules. For each such _sourceText_ and
   _hostDefined_, do
    1. If _sourceText_ is the source code of a script, then
        1. Perform EnqueueJob(**"ScriptJobs"**, ScriptEvaluationJob, «
           _sourceText_, _hostDefined_ »).
    2. Else,
        1. Assert: _sourceText_ is the source code of a module.
        2. Perform EnqueueJob(**"ScriptJobs"**, TopLevelModuleEvaluationJob, «
           _sourceText_, _hostDefined_ »).
3. Repeat,
    1. Suspend the running execution context and remove it from the execution
       context stack.
    2. Assert: The execution context stack is now empty.
    3. Let _nextQueue_ be a non-empty Job Queue chosen in an
       implementation-defined manner. If all Job Queues are empty, the result is
       implementation-defined.
    4. Let _nextPending_ be the PendingJob record at the front of _nextQueue_.
       Remove that record from nextQueue.
    5. Let _newContext_ be a new execution context.
    6. Set _newContext_'s Function to null.
    7. Set _newContext_'s Realm to _nextPending_.\[\[Realm]].
    8. Set _newContext_'s ScriptOrModule to _nextPending_.\[\[ScriptOrModule]].
    9. <ins>**NOTE: Absolutely unsure what to do here ¯\\\_(ツ)\_/¯** Set
       _newContext_'s LexicalEnvironment's
       EnvironmentRecord.\[\[CallerMetadata]] to !
       ObjectCreate(_nextPending_.\[\[CallerMetadata]]).</ins>
    10. Push _newContext_ onto the execution context stack; _newContext_ is now
        the running execution context.
    11. Perform any implementation or host environment defined job
        initialization using _nextPending_.
    12. Let _result_ be the result of performing the abstract operation named by
        _nextPending_.\[\[Job]] using the elements of
        _nextPending_.\[\[Arguments]] as its arguments.
    13. If _result_ is an abrupt completion, perform HostReportErrors(«
        _result_.\[\[Value]] »).
