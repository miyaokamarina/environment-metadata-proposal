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
file (e.g., if you transitioning from Vue to React or vice versa, and use mixed
React+Vue components).

Using this API, it is possible to intoroduce conventional framework-agnostic
symbols and pragmas, which will be used by frameworks.

The following code…

```jsx
import { jsx } from 'universal-jsx';

// Custom JSX factory and fragment symbols ↓
import { h, Fragment } from '@foo/bar';

jsx(h, Fragment);

const x = (
    <>
        Hello, world<em>‼</em>
    </>
);

const y = <a href='https://example.com/'>{x}</a>;
```

… transpiles to…

```javascript
import { jsx, $h, $Fragment } from 'universal-jsx';
import { h, Fragment } from '@foo/bar';

jsx(h, Fragment);

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

### Zones

Some functionality of withdrawn Zones proposal may be implemented using this
API.

**TODO:** Provide minimal example.

### Informative logs

**TODO:** Provide minimal example.

```javascript
// a.js
import { logger } from '@foo/bar';

logger.scope('a.js');

const doStuff = () => {
    // Do stuff.
    logger.log('Stuff done.');
};

element.addEventListener('click', () => {
    logger.scope('element@click');
    doStuff();
});

api.get('https://example.com/api/v1/init-stuff').then(() => {
    logger.scope('api :: init-stuff');
    doStuff();
});
```

Depending on how the logger is implemented, the `log` function called from
`doStuff` may print the following lines:

-   `[a.js] [element@click] Stuff done.` — when `doStuff` was called from event
    listenter.
-   `[a.js] [api :: init-stuff] Stuff done.` — when `doStuff` was called from
    the Promise callback.

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
