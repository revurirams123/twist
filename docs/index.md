# Twist - A JavaScript State Management Library

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Overview](#overview)
  - [Twist API Reference](#twist-api-reference)
    - [Stores](#stores)
    - [Actions](#actions)
    - [Connecting Model and View](#connecting-model-and-view)
    - [Stores and Two-Way Data Binding](#stores-and-two-way-data-binding)
    - [Mutable Stores](#mutable-stores)
    - [Derived State](#derived-state)
    - [Stores and Components](#stores-and-components)
  - [Middleware](#middleware)
    - [DevTools Middleware](#devtools-middleware)
    - [Thunk Middleware](#thunk-middleware)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Overview

The [Twist](https://github.com/adobe/twist) library provides a framework for managing state in your JavaScript application. It's influenced by [Redux](http://redux.js.org/) and [MobX](https://github.com/mobxjs/mobx), which are two of the popular state-management libraries for React. Twist makes it easy to:

* Encapsulate your application state in a **centralized location**, outside of the views/components.
* **Use data binding** to bind to the application state from your views/components, and have the view automatically update whenever the state changes.
* Easily **export/import your application state to/from JSON** - this is important for persistence, and server-side rendering.
* Easily **debug changes to the application state** using the excellent [Redux Devtools Extension](https://github.com/zalmoxisus/redux-devtools-extension).

Twist takes the idea of _observables_ and _mutable state_ from MobX, combined with the structure of _stores_ and _actions_ from Redux. When you use Twist with bindings to your UI layer (like React-Twist), it optimizes the rendering so that whenever the state changes, only the components that need to update will be re-rendered. While all state in Twist is _mutable_ under the hood, it also keeps the concept of _actions_ as the single mechanism for changing the state. By having a single pipeline that all state changes go through, it makes it easy to reason about how your application state changes over time, and gives you access to time-traveling debugging tools.


## Twist API Reference

There are two core notions in Twist, which are borrowed from Redux: **stores**, and **actions**.

### Stores

A **store** is a container for state. To define a store, simply make a new class using the `@Store` decorator. An instance of a store has properties for storing state, just like any JavaScript object. The difference is that in a store, we need to be explicit about how to serialize/deserialize the state to/from JSON. We do this using `@State.XXX` decorators on each of the properties (e.g. `@State.byVal`, `@State.byRef` etc). The `@State.XXX` decorators automatically make the properties observable, so there's no need to add `@Observable` in addition.

Stores are immutable by default, meaning that you can read the `@State.XXX` properties via data bindings from your view, but you can't directly modify them. If you want to modify one of these properties via two-way data binding, you need to create a local getter/setter as a proxy, so that you can dispatch an action in the setter (see the section on Actions below). Alternatively, you can make the store mutable, so that this mapping to an action happens automatically (see the section on [Mutable Stores](#mutable-stores)).

Here's a simple example of a store - notice that you can put one store inside another, using `@State.byRef`:

```javascript
@Store
class Address {
    @State.byVal state;
    @State.byNumberVal zipCode;
}

@Store
class User {
    @State.byVal title;
    @State.byVal firstName;
    @State.byVal lastName;

    @State.byRef(Address) address;
}
```

Every store has the methods `toJSON()` and `fromJSON()` methods, for serializing the store to/from a JSON. Additionally, `@Store` and `@State.XXX` prevent the state from being modified outside of an _action handler_ - which we'll now describe!

> Note: `toJSON()` doesn't return a JSON string, but rather a plain JavaScript that can be serialized to a string via `JSON.stringify`. This is to allow you to control the exact formatting (e.g. whether you want line breaks and indentation, etc).


### Actions

By itself, a store is fairly useless, because it can't be modified - the `@State.XXX` decorators prevent you from writing something like `...user.firstName = 'Bob'` directly in your code - it will throw an exception.

To allow state to be modified, you add **actions** to your store. The handler for an action is just a method, decorated with `@Action` - you can think of the `@Action` decorator as giving special permission to modify the state. Actions are invoked by calling `store.dispatch()` with the name of the action. Let's take a look at a simple example:

```javascript
@Store
class MyStore {

    @State.byRef(User) user;

    @Action SET_USER_TITLE(title) {
        this.user.title = title;
    }
}
```

To invoke the `SET_USER_TITLE` action in the above store, you would call:

```
store.dispatch('SET_USER_TITLE', title)
```

You can pass any number of parameters to an action, and they get passed in as the arguments to the handler.

We deviate from Redux here in two important ways. Firstly, the action handler (called a reducer in Redux) is _not_ a pure function - it mutates the state, rather than returning a new state. Secondly, we've put the logic for the actions in the store, with a separate function for each action (as opposed to a single function with a switch statement). We think this makes it a bit easier to read/write.

As we mentioned earlier, stores can be composed using `@State.byRef`. For example:

```javascript
@Store
class MyStore1 {
    ...
}

@Store
class MyStore2 {
    ...
}

@Store
class MyMainStore {
    @State.byRef(MyStore1) store1;
    @State.byRef(MyStore2) store2;
}
```

Dispatched actions get propagated to the sub-stores, so you can easily split up your application state into different pieces. Similarly, when you call `store.toJSON()` or `store.fromJSON()`, it will recurse through the sub-stores in addition to normal state.

By default, dispatched actions get propagated to all sub-stores of the store you dispatch to. The handler of the parent store will get called first, followed by the handlers of sub-stores. There are two ways to prevent the action from being propagated to sub-stores:

1. Return `true` from the action handler. This indicates that the action has been handled, and doesn't need to be propagated further.
2. Change `@Action` to `@Action({propagate: false})` - this tells Twist not to propagate the action (regardless of the return value of the action handler).


### Connecting Model and View

Hooking up a store (your model) to the view-layer of your application is easy. The top-level component (e.g. the main entry point for your application) creates a new store, and make it accessible to the rest of the application. If you're using React-Twist, you can do this using _scope_, as the following simple example illustrates:

```javascript
@Component({ fork: true })
class MyMainView {

    constructor() {
        super();

        // Assume we get INITIAL_STATE from the server, or from some local storage
        this.scope.store = new MyStore(INITIAL_STATE);
    }

    get store() {
        return this.scope.store;
    }

    knight() {
        this.store.dispatch('SET_USER_TITLE', 'Sir');
    }

    render() {
        return <g>
            <div>{ this.store.user.fullName }</div>
            <button on-click={ this.knight() }>Make Knight</button>
        </g>;
    }
}
```

Note that when you pass in `INITIAL_STATE` to the store's constructor, this actually maps onto a special `@@INIT` action that populates the state - this ensures that all state mutations happen through actions, even initialization!


### Stores and Two-Way Data Binding

Sometimes, having to dispatch actions to change the state can be a pain - it means that you can't directly bind to a value in a store, using two-way data binding, because it can't be updated outside of an action handler. For example, let's say you want to update the title in the `User` store, as the user is typing this into an input field, and have the store update automatically whenever the input field changes. You can still use two-way data-binding, but you need to introduce a getter and setter as a proxy, where the setter dispatches the appropriate action:

```javascript
@Component
class MyComponent {

    constructor() {
        super();
        this.scope.store = new MyStore;
    }

    get title() {
        return this.scope.store.user.title;
    }

    set title(newTitle) {
        this.scope.store.dispatch('SET_USER_TITLE', newTitle);
    }

    render() {
        return <input value={ this.title } />;
    }
}
```

### Mutable Stores

It would be easier if you could update the store directly, and so remove the need for the getter/setter. For this reason, Twist gives you the option of creating a **mutable store**. You can make a store mutable by passing a `mutable` option to the `@Store` decorator, as follows:

```javascript
@Store({mutable: true})
class User {
    @State.byVal title;
    @State.byVal firstName;
    @State.byVal lastName;

    @State.byRef(Address) address;
}
```

This lets you directly modify the store, so you can use two-way data binding to update it. This simplifies the above store to the following:

```javascript
@Component
class MyComponent {

    constructor() {
        super();
        this.scope.store = new MyStore;
    }

    render() {
        return <input value={ this.scope.store.user.title } />;
    }
}
```

The important thing to remember about mutable stores is that they _still use actions under the hood_. When you modify the store, an action gets created for you, meaning that it goes through the parent store's middleware. This means you can still use the redux-devtools-extension with mutable stores! The following diagram illustrates what's going on under the hood:

![Mutable Stores](./diagrams/implicit-action-dispatch.png?raw=true "Mutable Stores")

There are four stages to this:

1. When you change a mutable store, this gets converted to an action under the hood. The action name describes the property to set (or method to call), and the action payload is the value to assign to the property (or the arguments for the method).
2. (green arrows) The action is routed up the store hierarchy to the top-level store, and passed through its middleware. If you're using redux-devtools-extension, you'll see an action that looks like `'storeA/storeB/@property'` - this describes the path down the store hierarchy, and the property you were setting (in the case of a method, like calling `store.items.push(...)`, it would look something like `'storeA/storeB/@items.push()'`).
3. (red arrows) The action is routed back down to the store you changed - this is doing what's known as [source routing](https://en.wikipedia.org/wiki/Source_routing) in networking.
4. (blue arrow) The implicit action gets applied to the store. Unlike a normal action, it doesn't get propagated to any child stores of the target.

In a mutable store, you can change any of the `@State.XXX` properties, as well as arrays and maps created by `@State.byRefArray`/`@State.byRefMap`. The only limitation is that the payload of the action (the value you set, or the arguments to the method you call) must be _serializable_ - otherwise we wouldn't be able to record and replay the action. This means that while you _can_ call `this.items.reverse()` directly, you _can't_ call `this.items.sort(sortFunction)` because functions aren't serializable - you'd have to wrap it in an action.

If you don't specify the mutability of a store, it gets inherited from its parent. By default, stores are not mutable - but if you want to make all the stores mutable, just set the top-level store to mutable and the sub-stores will inherit this (you don't need to set `mutable: true` everywhere).

Note that right now, we only allow configuring mutability at a store-level granularity - we may add support for controlling mutability on a per-property level in future.


### Derived State

In the store example so far, we've only added fields for the raw data. More commonly, however, we want to display data in the view that's derived from the model. If this data is generally useful (e.g. used in more than one view), it makes sense to move it to the model. This can be done simply by adding a getter:

```javascript
@Store
class User {
    @State.byVal title;
    @State.byVal firstName;
    @State.byVal lastName;

    @State.byRef(Address) address;

    get fullName() {
        return (this.title ? this.title + ' ' : '') + this.firstName + ' ' + this.lastName;
    }
}
```

There's nothing special needed here - Twist's data binding will work across the getter, so whenever the raw model data changes, anyone bound to the getter will also update.

But what happens if we have an expensive operation in the getter? For example, we might be filtering on an array and doing some iteration and/or array allocations. If it's a plain getter, then _all_ data bindings that reference the getter in the application need to be re-evaluated whenever the raw state changes. That means we might be doing the same operation multiple times, which is inefficient. If this is the case, we can just mark the getter with the `@Cache` decorator:

```javascript
@Store
class MyState {

    ...

    @Cache
    get myProperty() {
        return this.expensiveComputation();
    }
}
```

This has the effect of introducing an intermediate property that watches the expression in the getter. When the state changes, the cache gets invalidated. Whenever the getter is invoked, it returns the cached value if there is one, or generates the cache. This means that if there are 10 places in the application that bind to this expression, it will only be re-evaluated once whenever the state changes.

Since `@Cache` is a tradeoff - it adds to memory usage, and a bit of overhead to the getter - only use it when the computation is sufficiently expensive and/or widely used to benefit from caching.


### Stores and Components

So far, we've described Twist from the perspective of an application, where you construct a single store for the entire application (which can be composed of other stores). In reality though, your application often consists of sub-components, some of which are not specific to your application, and may even be implemented by somebody else as a shared component.

If we continue with our example of the `User` store, let's imagine that we have a shared component that needs information about the logged-in user. If we already have a `User` store in our main store, then it doesn't make sense for the shared component to create another - there would be duplicate effort in populating the store (e.g. via extra network calls), and we'd have two user stores that could get out of sync with one another. So, let's pass in the store to the component:

```javascript
@Component({fork: true})
class UserProfileView {

    @Attribute userStore;

    render() {
        return <div>
            Name: { this.userStore.name }
            ...
        </div>;
    }
}

...

<UserProfileView userStore={ this.scope.store.user } />
```

Binding to data from the store is easy, but what about making changes to the store? You want all changes to go through the top-level store, so that it passes through a common middleware (e.g. if you're using redux-devtools-extension), but a shared components shouldn't need to know about how your application store is structured - it only cares about once small piece of the store.

To make life easy in cases like this, Twist lets you dispatch directly to a sub-store, and under the hood routes the action via the top-level store. The following diagram shows how this works:

![Dispatching to Child Stores](./diagrams/sub-store-dispatch.png?raw=true "Dispatching to Child Stores")

There are three stages when you call `dispatch` on a child store:

1. (green arrows) The action is routed up the store hierarchy to the top-level store, and passed through its middleware. If you're using redux-devtools-extension, you'll see an action that looks like `'storeA/storeB/ACTION_NAME'` - this encodes the path down the store hierarchy into the action name.
2. (red arrows) The action is routed back down to the store you changed - this is doing what's known as [source routing](https://en.wikipedia.org/wiki/Source_routing) in networking.
3. (blue arrow) The action gets dispatched to the store. Just like if you dispatched an action to the top-level store, this gets propagated to child stores (essentially, broadcast down the tree), but only starting from the sub-store you dispatched to. Remember that you can prevent propagation by returning `true` in the action handler.

This makes it easy to pass stores to reusable components - they just need to agree on the interface that the store provides.

One thing to be careful of, is that you can't compose the same store inside of two parents. For example, the following is not allowed:

```javascript
@Store
class A {
}

@Store
class B {
    @State.byRef(A) a;

    @Action SET(a) {
        this.a = a;
    }
}

var a = new A();
var b1 = new B();
var b2 = new B();

// This succeeds, turning a into a sub-store of b1:
b1.dispatch('SET', a);

// This fails - since a is already a sub-store of b1, it can't also be a sub-store of b2:
b2.dispatch('SET', a);
```

There are many good reasons for requiring the structure of your store hierarchy to strictly be a _tree_. The most obvious reasons are (a) it leads to a lot of confusing cases, like actions being handled multiple times by the same store, and not knowing which top-level store to pass actions through, and (b) links aren't preserved when you serialize to/from JSON.


## Middleware

When you create a store, you can also pass in a _middleware_ argument to the constructor. This is just an array of functions of the form:

```javascript
function (store, action, payload, done) {
    // Here you can intercept the action (e.g. to support async actions), or you can simply log what goes by (e.g. for devtools).
    // To continue to the next action, call the done() callback - otherwise, the action will stop being dispatched.
    // To modify the action, call done(newAction, newPayload) - if you don't provide an argument to done(), it'll continue with the action unchanged.
    // Note that payload is an array of arguments (because actions may take multiple arguments).
}
```

You would change the initialization of the store to:

```javascript
this.scope.store = new MyStore(INITIAL_STATE, [Middleware1, Middleware2]);
```

Twist comes bundled with middleware for the most common use cases: debugging, and asynchronous actions.

### Thunk Middleware

This allows you to have _asynchronous_ actions. The action handlers on a store that you define with `@Action` must be synchronous, but there's nothing stopping you from composing more complex actions that are asynchronous - so long as all the state mutations are described in terms of simple synchronous actions.

To make this easy, the _thunk middleware_ lets you pass in a function as an action. This is enabled by default. Here's an example:

```javascript
var store = new MyStore(INITIAL_STATE);

var DELAYED_SET_USER_TITLE = function (store, payload) {
    setTimeout(() => store.dispatch('SET_USER_TITLE', payload.title), payload.delay);
};

store.dispatch(DELAYED_SET_USER_TITLE, { title: 'Dr.', delay: 1000 });
```

A common use case for asynchronous actions is when fetching data from a REST API. Note that the function you pass in can dispatch as many actions as you like, including other asynchronous actions.

You can also add an asynchronous action to your store, using `@Action({async: true})` - this allows us to write the above as:

```javascript
@Store
class MyStore {

    @State.byRef(User) user;

    @Action SET_USER_TITLE(title) {
        this.user.title = title;
    }

    @Action({async: true}) DELAYED_SET_USER_TITLE(payload) {
        setTimeout(() => this.dispatch('SET_USER_TITLE', payload.title), payload.delay);
    }
}

var store = new MyStore(INITIAL_STATE);
store.dispatch('DELAYED_SET_USER_TITLE', { title: 'Dr.', delay: 1000 });
```

Note that asynchronous actions can in turn dispatch both synchronous and asynchronous actions, but synchronous actions _cannot_ dispatch asynchronous actions (for obvious reasons).

As a consequence, if you mark an action handler as `@Action({async: true})`, then it can _only_ be invoked by dispatching it directly on that store. If you dispatch an action with this name to a parent store, then it will _not_ execute matching asynchronous actions in child stores, as it propagates. This is because only synchronous actions can propagate, and synchronous actions cannot dispatch asynchronous actions.


### DevTools Middleware

[Redux-devtools-extension](https://github.com/zalmoxisus/redux-devtools-extension) is a browser plugin for Chrome/Firefox (and also Electron), that was built for debugging Redux applications. Twist comes bundled with middleware for connecting to this, so it's really easy to debug your store. Just pass in the middleware when creating your store:

```javascript
import { remoteDevMiddleware } from '@twist/core';

var store = new MyStore(INITIAL_STATE, [remoteDevMiddleware]);
```

If you don't already have the browser plugin, you can install it from [here](https://github.com/zalmoxisus/redux-devtools-extension).

The plugin records all the actions and changes to the state of your store, and lets you time travel to previous states, toggle actions, and do diffing on the state. This is all connected live to your application, so you can easily see how changes in the state get reflected in the view. You can also dispatch actions from the plugin, and save and restore the state, so you can pass it to somebody else, or include it in a bug report.

We recommend only using the remotedev middleware in your development environment / builds - not for production builds. This is because (a) you probably don't want your users to be able to debug your application, and (b) it adds a fair amount of overhead if the redux-devtools-extension is installed, since it records all changes to the state.


### Protector Middleware

When you define synchronous actions in Twist, it's really important that they are _actually_ synchronous. A common mistake is to do something asynchronous, like a network call, or setting a timeout. The problem with this, is that it makes the changes to the store in response to an action non-deterministic. If you're using the [DevTools Middleware](#devtools-middleware), this poses a particular problem, because it needs to be able to replay synchronous actions as you move forwards and backwards in time.

To help detect such common mistakes, Twist comes with _protector middleware_ that automatically blocks certain common asynchronous operations from inside of synchronous action handlers - for example, `setTimeout`, `setInterval`, and the use of promises. If you need to do something asynchronous, just make the action asynchronous (`@Action({async: true})`)! Since this has to replace some of the browser globals to achieve this, we recommend only using the protector middleware in your development environment / builds - not for production builds.

To use this middleware, just pass it in when creating your store:

```javascript
import { protectorMiddleware } from '@twist/core';

var store = new MyStore(INITIAL_STATE, [protectorMiddleware]);
```