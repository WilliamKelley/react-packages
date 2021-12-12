# react-meteor-data

An integration package between [React](https://reactjs.org/) and [Tracker](https://docs.meteor.com/api/tracker.html), Meteor's reactive data system.

## Table of Contents

- [Installation](#installation)
  - [Changelog](#changelog)
- [Usage](#usage)
  - [Versions summary](#versions-summary)
  - [`useTracker`](#usetracker)
    - [Basic usage](#basic-usage)
    - [With dependencies](#with-dependencies)
    - [Advanced usage](#advanced-usage)
  - [`useSubscribe`](#usesubscribe)
  - [`useFind`](#usefind)
  - [`withTracker`](#withtracker)
    - [Basic usage](#basic-usage-1)
    - [Advanced usage](#advanced-usage-1)
- [Appendix](#appendix)
  - [On ESLint](#on-eslint)
    - [Exhaustive dependencies](#enforcing-exhaustive-dependencies)
  - [On TypeScript Interfaces](#on-typescript-interfaces)
    - [Reactive function](#ireactivefnt)
    - [Skip update](#iskipupdatet)

## Installation

To install the package, use `meteor add`:

```bash
meteor add react-meteor-data
```

You'll also need to install `react` if you have not already:

```bash
meteor npm install react
```

### Changelog

For recent changes, read the [CHANGELOG](./CHANGELOG.md).

## Usage

This package provides two ways to use Tracker reactive data in your React components: `withTracker` and `useTracker`. The `withTracker` function is a [higher-order component (HOC)](https://reactjs.org/docs/higher-order-components.html) that can be used with all components, functional or class-based. The `useTracker` function is a React hook, introduced in version 2.0.0, and embraces the [benefits of hooks](https://reactjs.org/docs/hooks-faq.html). Like all React hooks, it can only be used in functional components, not in class components.

_Note:_ It is not necessary to rewrite existing applications to use the `useTracker` hook instead of the existing `withTracker` HOC.

### Versions summary

|  | v1 | v2 | v2.4 |
| --- | --- | --- | --- |
| React | `>=15.3.0` | `>=16.8.0` | (same as v2) |
| `withTracker` | ✅ | ✅ | ✅ |
| `useTracker` | ❌ | ✅ | ✅ |
| └─ `useSubscribe` | ❌ | ❌ | ✅ |
| └─ `useFind` | ❌ | ❌ | ✅ |

### `useTracker(...)`

Get the value of a Tracker reactive function in your React "function components." The reactive function will get re-run whenever its reactive inputs change, and the component will re-render with the new value.

The hook manages its own state and causes re-renders when necessary. There is no need to call React state setters from inside your reactive function (`reactiveFn`). Instead, return values and assign them to variables, directly. When the reactive function updates, the variables will be updated, and the React component will re-render.

#### TypeScript Signature

_(+1 overload)_

```ts
function useTracker <T = any>(reactiveFn: IReactiveFn<T>, skipUpdate?: ISkipUpdate<T>): T;
function useTracker <T = any>(reactiveFn: IReactiveFn<T>, deps?: DependencyList, skipUpdate?: ISkipUpdate<T>): T;
```

#### Arguments

| Argument | Type | Required | Description |
| --- | --- | --- | --- |
| reactiveFn | [`IReactiveFn<T>`](#ireactivefnt) | Yes | A Tracker reactive function (receives the current computation). |
| deps | `React.DependencyList` | No | An optional array of "dependencies" of the reactive function. This is very similar to how the `deps` argument for [React's built-in `useEffect`, `useCallback` or `useMemo` hooks](https://reactjs.org/docs/hooks-reference.html) work. |
| skipUpdate | [`ISkipUpdate<T>`](#iskipupdatet) | No | 


#### Basic usage

Simply pass it a reactive function, with no further fuss. This is the preferred configuration in many cases.

`useTracker(reactiveFn)`

```jsx
import React from 'react';
import { useTracker } from 'meteor/react-meteor-data';

// React function component.
function Foo() {
  // Get the logged-in user record (a reactive data source, see https://docs.meteor.com/api/accounts.html#Meteor-user )
  const currentUser = useTracker(() => Meteor.user());
  
  return <h1>Hello {currentUser.username}</h1>;
}
```

#### With dependencies

You can pass an optional deps array as a second value. When provided, the computation will be retained, and reactive updates after the first run will run asynchronously from the react render execution frame. This array typically includes all variables from the outer scope "captured" in the closure passed as the 1st argument. For example, the value of a prop used in a subscription or a minimongo query; see example below.

`useTracker(reactiveFn, deps)` 

This should be considered a low level optimization step for cases where your computations are somewhat long running -- like a complex minimongo query. In many cases it's safe, and even preferred, to omit deps and allow the computation to run synchronously with render.

```js
import { useTracker } from 'meteor/react-meteor-data';

function Foo({ listId }) {
  // This computation uses no value from the outer scope,
  // and thus does not needs to pass a 'deps' argument.
  // However, we can optimize the use of the computation
  // by providing an empty deps array. With it, the
  // computation will be retained instead of torn down and
  // rebuilt on every render. useTracker will produce the
  // same results either way.
  const currentUser = useTracker(() => Meteor.user(), []);

  // The following two computations both depend on the
  // listId prop. When deps are specified, the computation
  // will be retained.
  const isListLoading = useTracker(() => {
    // Note that this subscription will get cleaned up
    // when your component is unmounted or deps change.
    const handle = Meteor.subscribe('todoList', listId);
    return !handle.ready();
  }, [listId]);
  
  const tasks = useTracker(() => {
    return Tasks.find({ listId }).fetch();
  }, [listId]);

  return (
    <h1>Hello {currentUser.username}</h1>
    {isListLoading ? (
        <div>Loading</div>
      ) : (
        <div>
          Here is the Todo list {listId}:
          <ul>
            {tasks.map(task => (
              <li key={task._id}>{task.label}</li>
            ))}
          </ul>
        </div>
      )}
  );
}
```

**Note:** the [eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks) package provides ESLint hints to help detect missing values in the `deps` argument of React built-in hooks. It can be configured to also validate the `deps` argument of the `useTracker` hook or some other hooks, with the following `eslintrc` config:

```
"react-hooks/exhaustive-deps": ["warn", { "additionalHooks": "useTracker|useSomeOtherHook|..." }]
```

#### `useTracker(reactiveFn, deps, skipUpdate)` or `useTracker(reactiveFn, skipUpdate)`

You may optionally pass a function as a second or third argument. The `skipUpdate` function can evaluate the return value of `reactiveFn` for changes, and control re-renders in sensitive cases. *Note:* This is not meant to be used iwth a deep compare (even fast-deep-equals), as in many cases that may actually lead to worse performance than allowing React to do it's thing. But as an example, you could use this to compare an `updatedAt` field between updates, or a subset of specific fields, if you aren't using the entire document in a subscription. As always with any optimization, measure first, then optimize second. Make sure you really need this before implementing it.

Arguments:
- `reactiveFn`
- `deps?` - optional - you may omit this, or pass a "falsy" value.
- `skipUpdate` - A function which receives two arguments: `(prev, next) => (prev === next)`. `prev` and `next` will match the type or data shape as that returned by `reactiveFn`. Note: A return value of `true` means the update will be "skipped". `false` means re-render will occur as normal. So the function should be looking for equivalence.

```jsx
import { useTracker } from 'meteor/react-meteor-data';

// React function component.
function Foo({ listId }) {
  const tasks = useTracker(
    () => Tasks.find({ listId }).fetch(), [listId],
    (prev, next) => {
      // prev and next will match the type returned by the reactiveFn
      return prev.every((doc, i) => (
        doc._id === next[i] && doc.updatedAt === next[i]
      )) && prev.length === next.length;
    }
  );

  return (
    <h1>Hello {currentUser.username}</h1>
    <div>
      Here is the Todo list {listId}:
      <ul>
        {tasks.map(task => (
          <li key={task._id}>{task.label}</li>
        ))}
      </ul>
    </div>
  );
}
```

#### `withTracker(reactiveFn)` higher-order component

You can use the `withTracker` HOC to wrap your components and pass them additional props values from a Tracker reactive function. The reactive function will get re-run whenever its reactive inputs change, and the wrapped component will re-render with the new values for the additional props.

Arguments:
- `reactiveFn`: a Tracker reactive function, getting the props as a parameter, and returning an object of additional props to pass to the wrapped component.

```js
import { withTracker } from 'meteor/react-meteor-data';

// React component (function or class).
function Foo({ listId, currentUser, listLoading, tasks }) {
  return (
    <h1>Hello {currentUser.username}</h1>
    {listLoading ?
      <div>Loading</div> :
      <div>
        Here is the Todo list {listId}:
        <ul>{tasks.map(task => <li key={task._id}>{task.label}</li>)}</ul>
      </div}
  );
}

export default withTracker(({ listId }) => {
  // Do all your reactive data access in this function.
  // Note that this subscription will get cleaned up when your component is unmounted
  const handle = Meteor.subscribe('todoList', listId);

  return {
    currentUser: Meteor.user(),
    listLoading: !handle.ready(),
    tasks: Tasks.find({ listId }).fetch(),
  };
})(Foo);
```

The returned component will, when rendered, render `Foo` (the "lower-order" component) with its provided props in addition to the result of the reactive function. So `Foo` will receive `{ listId }` (provided by its parent) as well as `{ currentUser, listLoading, tasks }` (added by the `withTracker` HOC).

For more information, see the [React article](http://guide.meteor.com/react.html) in the Meteor Guide.

#### `withTracker({ reactiveFn, pure, skipUpdate })` advanced container config

The `withTracker` HOC can receive a config object instead of a simple reactive function.

- `getMeteorData` - The `reactiveFn`.
- `pure` - `true` by default. Causes the resulting Container to be wrapped with React's `memo()`.
- `skipUpdate` - A function which receives two arguments: `(prev, next) => (prev === next)`. `prev` and `next` will match the type or data shape as that returned by `reactiveFn`. Note: A return value of `true` means the update will be "skipped". `false` means re-render will occur as normal. So the function should be looking for equivalence.

```js
import { withTracker } from 'meteor/react-meteor-data';

// React component (function or class).
function Foo({ listId, currentUser, listLoading, tasks }) {
  return (
    <h1>Hello {currentUser.username}</h1>
    {listLoading ?
      <div>Loading</div> :
      <div>
        Here is the Todo list {listId}:
        <ul>{tasks.map(task => <li key={task._id}>{task.label}</li>)}</ul>
      </div}
  );
}

export default withTracker({
  getMeteorData ({ listId }) {
    // Do all your reactive data access in this function.
    // Note that this subscription will get cleaned up when your component is unmounted
    const handle = Meteor.subscribe('todoList', listId);

    return {
      currentUser: Meteor.user(),
      listLoading: !handle.ready(),
      tasks: Tasks.find({ listId }).fetch(),
    };
  },
  pure: true,
  skipUpdate (prev, next) {
    // prev and next will match the shape returned by the reactiveFn
    return (
      prev.currentUser?._id === next.currentUser?._id
    ) && (
      prev.listLoading === next.listLoading
    ) && (
      prev.tasks.every((doc, i) => (
        doc._id === next[i] && doc.updatedAt === next[i]
      ))
      && prev.tasks.length === next.tasks.length
    );
  }
})(Foo);
```

#### `useSubscribe(subName, ...args)` A convenient wrapper for subscriptions

`useSubscribe` is a convenient short hand for setting up a subscription. It is particularly useful when working with `useFind`, which should NOT be used for setting up subscriptions. At its core, it is a very simple wrapper around `useTracker` (with no deps) to create the subscription in a safe way, and allows you to avoid some of the cerimony around defining a factory and defining deps. Just pass the name of your subscription, and your arguments.

`useSubscribe` returns an `isLoading` function. You can call `isLoading()` to react to changes in the subscription's loading state. The `isLoading` function will both return the loading state of the subscription, and set up a reactivity for the loading state change. If you don't call this function, no re-render will occur when the loading state changes.

```jsx
// Note: isLoading is a function!
const isLoading = useSubscribe('posts', groupId);
const posts = useFind(() => Posts.find({ groupId }), [groupId]);

if (isLoading()) {
  return <Loading />
} else {
  return <ul>
    {posts.map(post => <li key={post._id}>{post.title}</li>)}
  </ul>
}
```

If you want to conditionally subscribe, you can set the `name` field (the first argument) to a falsy value to bypass the subscription.

```jsx
const needsData = false;
const isLoading = useSubscribe(needsData ? "my-pub" : null);

// When a subscription is not used, isLoading() will always return false
```

#### `useFind(cursorFactory, deps)` Accellerate your lists

The `useFind` hook can substantially speed up the rendering (and rerendering) of lists coming from mongo queries (subscriptions). It does this by controlling document object references. By providing a highly tailored cursor management within the hook, using the `Cursor.observe` API, `useFind` carefully updates only the object references changed during a DDP update. This approach allows a tighter use of core React tools and philosophies to turbo charge your list renders. It is a very different approach from the more general purpose `useTracker`, and it requires a bit more set up. A notable difference is that you should NOT call `.fetch()`. `useFind` requires its factory to return a `Mongo.Cursor` object. You may also return `null`, if you want to conditionally set up the Cursor.

Here is an example in code:

```jsx
import React, { memo } from 'react'
import { useFind } from 'meteor/react-meteor-data'
import TestDocs from '/imports/api/collections/TestDocs'

// Memoize the list item
const ListItem = memo(({doc}) => {
  return (
    <li>{doc.id},{doc.updated}</li>
  )
})

const Test = () => {
  const docs = useFind(() => TestDocs.find(), [])
  return (
    <ul>
      {docs.map(doc =>
        <ListItem key={doc.id} doc={doc} />
      )}
    </ul>
  )
}

// Later on, update a single document - notice only that single component is updated in the DOM
TestDocs.update({ id: 2 }, { $inc: { someProp: 1 } })
```

If you want to conditionally call the find method based on some props configuration or anything else, return `null` from the factory.

```jsx
const docs = useFind(() => {
  if (props.skip) {
    return null
  }
  return TestDocs.find()
}, [])
```

### Concurrent Mode, Suspense and Error Boundaries

There are some additional considerations to keep in mind when using Concurrent Mode, Suspense and Error Boundaries, as each of these can cause React to cancel and discard (toss) a render, including the result of the first run of your reactive function. One of the things React developers often stress is that we should not create "side-effects" directly in the render method or in functional components. There are a number of good reasons for this, including allowing the React runtime to cancel renders. Limiting the use of side-effects allows features such as concurrent mode, suspense and error boundaries to work deterministically, without leaking memory or creating rogue processes. Care should be taken to avoid side effects in your reactive function for these reasons. (Note: this caution does not apply to Meteor specific side-effects like subscriptions, since those will be automatically cleaned up when `useTracker`'s computation is disposed.)

Ideally, side-effects such as creating a Meteor computation would be done in `useEffect`. However, this is problematic for Meteor, which mixes an initial data query with setting up the computation to watch those data sources all in one initial run. If we wait to do that in `useEffect`, we'll end up rendering a minimum of 2 times (and using hacks for the first one) for every component which uses `useTracker` or `withTracker`, or not running at all in the initial render and still requiring a minimum of 2 renders, and complicating the API.

To work around this and keep things running fast, we are creating the computation in the render method directly, and doing a number of checks later in `useEffect` to make sure we keep that computation fresh and everything up to date, while also making sure to clean things up if we detect the render has been tossed. For the most part, this should all be transparent.

The important thing to understand is that your reactive function can be initially called more than once for a single render, because sometimes the work will be tossed. Additionally, `useTracker` will not call your reactive function reactively until the render is committed (until `useEffect` runs). If you have a particularly fast changing data source, this is worth understanding. With this very short possible suspension, there are checks in place to make sure the eventual result is always up to date with the current state of the reactive function. Once the render is "committed", and the component mounted, the computation is kept running, and everything will run as expected.

### Version compatibility notes

- `react-meteor-data` v2.x :
  - `useTracker` hook + `withTracker` HOC
  - Requires React `^16.8`.
  - Implementation is compatible with "React Suspense", concurrent mode and error boundaries.
  - The `withTracker` HOC is strictly backwards-compatible with the one provided in v1.x, the major version number is only motivated by the bump of React version requirement. Provided a compatible React version, existing Meteor apps leveraging the `withTracker` HOC can freely upgrade from v1.x to v2.x, and gain compatibility with future React versions.
  - The previously deprecated `createContainer` has been removed.

- `react-meteor-data` v0.x :
  - `withTracker` HOC (+ `createContainer`, kept for backwards compatibility with early v0.x releases)
  - Requires React `^15.3` or `^16.0`.
  - Implementation relies on React lifecycle methods (`componentWillMount` / `componentWillUpdate`) that are [marked for deprecation in future React versions](https://reactjs.org/blog/2018/03/29/react-v-16-3.html#component-lifecycle-changes) ("React Suspense").
