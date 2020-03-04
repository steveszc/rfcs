- Start Date: 2020-01-10
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/580
- Tracking: (leave this empty)

# Destroyables

## Summary

Adds an API for registering destroyables and destructors with Ember's built in
destruction hierarchy.

```js
class MyComponent extends Component {
  constructor() {
    let timeoutId = setTimeout(() => console.log('hello'), 1000);

    registerDestructor(this, () => clearTimeout(timeoutId));
  }
}
```

The API will also enable users to create and manage their own destroyables, and
associate them with a parent destroyable.

```js
class TimeoutManager {
  constructor(parent, fn, timeout = 1000) {
    let timeoutId = setTimeout(fn, timeout);

    associateDestroyableChild(parent, this);
    registerDestructor(this, () => clearTimeout(timeoutId))
  }
}

class MyComponent extends Component {
  manager = new TimeoutManager(this, () => console.log('hello'));
}
```

## Motivation

Ember manages the lifecycles and lifetimes of many built in constructs, such as
components, and does so in a hierarchical way - when a parent component is
destroyed, all of its children are destroyed as well. This is a well established
software pattern that is useful for many applications, and there are a variety
of libraries, such as [ember-lifeline](https://github.com/ember-lifeline/ember-lifeline)
and [ember-concurrency](https://github.com/machty/ember-concurrency), that would
benefit from having a way to extend this hierarchy, adding their own "children"
that are cleaned up whenever their parents are removed.

Historically, Ember has exposed this cleanup lifecycle via _hooks_, such as the
`willDestroy` hook on components. However, methods like these have a number of
downsides:

1. Since they are named, they can have collisions with other properties. This is
   historically what led to the actions hash on classic components, in order to
   avoid collisions between actions named `destroy` and the `destroy` lifecyle
   hook.
2. On a related note, relying on property names means that all framework classes
   _must_ implement the `willDestroy` function (or another name), making it very
   difficult to change APIs in the future.
3. Methods are difficult for _libraries_ to instrument. For instance,
   `ember-concurrency` currently replaces the `willDestroy` method on any class
   with a task, with logic that looks similar to:

   ```js
   let PATCHED = new WeakSet();

   function patchWillDestroy(obj) {
     if (PATCHED.has(obj)) return;

     let oldWillDestroy = obj.willDestroy;

     obj.willDestroy = function() {
       if (oldWillDestroy) oldWillDestroy.call(this);

       teardownTasks(this);
     }

     PATCHED.add(obj);
   }
   ```

   This logic becomes especially convoluted if _multiple_ libraries are
   attempting to patch `willDestroy` in this way.

4. Finally, since this isn't a standard, it's difficult to add _layers_ of new
   destroyable values that can interoperate with one another. For instance,
   there is no way for `ember-concurrency` to know how to destroy tasks on
   non-framework classes that users may have added themselves.

This RFC proposes a streamlined API that disconnects the exact implementation
from any interface, allows for multiple destructors per-destroyable, and
maximizes interoperability in general.

## Detailed design

The API consists of 6 main functions, imported from `@ember/destroyable`:

```ts
declare function associateDestroyableChild(parent: object, child: object): void;
declare function registerDestructor(destroyable: object, destructor: () => void): void;
declare function unregisterDestructor(destroyable: object, destructor: () => void): void;

declare function destroy(destroyable: object): void;
declare function isDestroying(destroyable: object): boolean;
declare function isDestroyed(destroyable: object): boolean;
```

In addition, there is a debug-only mode function used for testing:

```ts
declare function assertDestroyablesDestroyed(): void;
```

For the remainder of this RFC, the terms "destroyable" and "destroyable object"
will be used to mean any object which is a valid `WeakMap` key
(e.g. `typeof obj === 'object' || typeof obj === 'function'`). Any JS object
that fulfills this property can be used with this system.

#### `associateDestroyableChild`

This function is used to associate a destroyable object with a parent. When the
parent is destroyed, all registered children will also be destroyed.

- Attempting to associate a parent or child that has already been destroyed
  should throw an error.

##### Multiple Inheritance

Attempting to associate a child to multiple parents should currently throw an
error. This could be changed in the future, but for the time being multiple
inheritance of destructors is tricky and not scoped in. Instead, users can add
destructors to accomplish this goal:

```js
let parent1 = {}, parent2 = {}, child = {};

registerDestructor(parent1, () => destroy(child));
registerDestructor(parent2, () => destroy(child));
```

The exact timing semantics here will be a bit different, but for most use cases
this should be fine. If we find that it would be useful to have multiple
inheritance baked in in the future, it can be added in a followup RFC.

#### `registerDestructor`

Receives a destroyable object and a destructor function, and associates the
function with it. When the destroyable is destroyed with `destroy`, or when its
parent is destroyed, the destructor function will be called. Multiple
destructors can be associated with a given destroyable, and they can be
associated over time, allowing libraries like `ember-lifeline` to dynamically
add destructors as needed.

- Registering a destructor on a destroyed object should throw an error.
- Attempting to register the same destructor multiple times should throw an
  error.

#### `unregisterDestructor`

Receives a destroyable and a destructor function, and de-associates the
destructor from the destroyable.

- Calling `unregisterDestructor` on a destroyed object should throw an error.
- Calling `unregisterDestructor` with a destructor that is not associated with
  the object should throw an error.

#### `destroy`

`destroy` initiates the destruction of a destroyable object. It runs all
associated destructors, and then destroys all children recursively.

Destruction via `destroy()` follows these steps:

1. Mark the destroyable such that `isDestroying(destroyable)` returns `true`
2. Schedule calling the destroyable's destructors
3. Call `destroy()` on each of the destroyable's associated children
4. Schedule setting destroyable such that `isDestroyed(destroyable)` returns `true`

This algorithm results in the entire tree of destroyables being first marked as
destroying, then having all of their destructors called, and finally all being
marked as `isDestroyed`. There won't be any in between states where some items
are marked as `isDestroying` while destroying, while others are not.

Calling `destroy` multiple times on the same destroyable is safe. It will not
throw an error, and will not take any further action.

Calling `destroy` with a destroyable that has no destructors or associated children
will not throw an error, and will do nothing.

#### `isDestroying`

Receives a destroyable, and returns `true` if the destroyable has begun
destroying. Otherwise returns false.

#### `isDestroyed`

Receives a destroyable, and returns `true` if the destroyable has finished
destroying. Otherwise returns false.

#### `assertDestroyablesDestroyed`

This function asserts that all objects which have associated destructors or
associated children have been destroyed at the time it is called. It is meant to
be a low level hook that testing frameworks like `ember-qunit` and `ember-mocha`
can use to hook into and validate that all destroyables have in fact been
destroyed.

### Built In Destroyables

The root destroyable of an Ember application will be the instance of the owner.
All framework managed classes are destroyables, including:

* Components
* Services
* Routes
* Controllers
* Helpers
* Modifiers

Any future classes that are added and have a container managed lifecycle should
also be marked as destroyables.

## How we teach this

This is a generally low level API, meant for usage in the framework and in
addons. It should be taught primarily through API documentation, and potentially
through an in-depth guide.

## Drawbacks

- Adds another destruction API which may conflict with the existing destruction
  hooks. Since this is a low-level API, it shouldn't be too problematic - most
  users will be guided toward using the standard lifecycle hooks, and this API
  will exist for libraries like `ember-concurrency` and `ember-lifeline`.

## Alternatives

- Continue using existing lifecycle hooks for public API, and don't provide an
  independent API.

## Unresolved questions

- Should `willDestroy` and `didDestroy` (e.g. mark-and-sweep style semantics) be
  exposed?
