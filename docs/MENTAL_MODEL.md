# Mental Model

## Overview

Qwik is a different kind of framework. Qwik stores all of the application information in the DOM/HTML rather than in Javascript heap. The result is that the applications written in Qwik can be serialized and resumed where they left off. Qwik's design decisions are governed by:

- **Serializability:** Ability of the application to be serialized into HTML at any time and resumed later. (This supports resumability and server-side-rendering.)
- **Resumability:** Ability of the application to continue where it left off when it was serialized. (This is needed to have fast application startup times independent of application size.)
- **Out-of-order component rehydration:** Ability to rehydrate components on an as-needed basis regardless of where they are in the DOM hierarchy. (This is needed to only rehydrate the components that the user is interacting with (or components that need to be updated) rather than all of the components on the page.)
- **Fine-grained lazy-loading:** Ability to only download the piece of code that is needed to achieve the current task. (Minimize the amount of code that needs to be downloaded into the browser, ensuring fast startup and responsiveness even on slow connections.)
- **Startup Performance:** Have a constant startup time regardless of the size of the application. (No matter the application's size, the startup time should be constant to ensure a consistent user experience.)

Because of the above goals, some design decisions may feel foreign compared to the current generation of "heap-centric" frameworks. However, the design decisions are necessary to achieve the above objectives, ultimately leading to blazingly fast application startup on desktop and mobile devices.

## Component & Entities

Qwik applications consist of `Component`s and `Entity`s.

- **Component:** Qwik applications UIs are built from a collection of Qwik `Component`s. `Component`s can communicate with each other only through `Props`, `Entity`s, and by listening to or emitting/broadcasting `Event`s. (See [reactivity](./REACTIVITY.md)). No other form of communication is allowed as it would go against the goals outlined above.
- **Entity:** `Entity`s represent application data or shared behavior between `Component`s and the application backend. `Entity`s do not have UI representation and must be injected into a `Component` to have their data rendered. `Entity`s can perform communication with the server or other external resources.

A Qwik application is a graph of `Component`s and `Entity`s that define data flow. The Qwik framework will hydrate `Component`s/`Entity`s on an as-needed basis to achieve the desired outcome. To do this, the Qwik framework must understand the data flow between `Component`s and `Entity`s to know which `Component` or `Entity` needs to be rehydrated and when. Most other frameworks do not have this understanding and need to re-render the whole application on any data update resulting in too much code that must be downloaded. (See [lazy loading](./LAZY_LOADING.md).)

### Example

```
TodoComponent   <---------- TodoDataEntity
     |                           |
     +-> ItemComponent  <-----   +--> ItemDataEntity
```

Above is a `Component`/`Entity` dependency graph of a hypothetical application. in Qwik, relationships between `Component`s and `Entity`s are expressed in the DOM/HTML. Depending on which object updates, Qwik understands which other `Component`s need to be rehydrated and re-rendered as well. (Without this understanding, Qwik would have to re-render the whole application indiscriminately.)

## Serializability of Components & Entities.

`Component`s and `Entity`s must be serializable so that they can be sent over HTML and resumed later. This serializability requirement is necessary to have resumable applications.

A `Component` or `Entity` consist of:

- **`Props`:** These are a hash of key/value strings (`{[key:string]:string}`), which tell Qwik about the `Component` or `Entity`.
  - **`Component`:** These are just the DOM attributes of the `Component`'s host element. If the attributes change, then the `Component` gets notified. These `Props` are serialized into the [host element](./HOST_ELEMENT.md).
  - **`Entity`:** The `Props` for a `Entity`s are usually serialized into a `EntityKey`, which uniquely identifies a entity instance. The `EntityKey` is constant for the lifetime of the entity. (For example `issue:org123:proj456:789` might identify an issue with id `789` in `org123/proj456`.)
- **`State`:** A `Component` or `Entity` have a `State` which is a `JSON` serializable object. If an application is dehydrated then then the `State` is serialized into the DOM/HTML so that it can be quickly rehydrated on the client.
- **Transient State:** is state that is only stored inside the instance of a `Component` or `Entity`. This state can include references to any other object. However, the transient state will not be serialized and therefore anything stored on the instance as transient state will have to be recomputed on rehydration.

Example:

```typescript
// Serialized into a entity Key.
interface UserProps {
  id: string;
}

// Serialized into HTML/DOM as JSON.
interface User {
  fullName: string;
  age: number;
}

// Transient instance. Will be recreated on re-hydration.
class UserEntity extends Entity<UserProps, User> {
  static $type = 'User';
  static $qrl = QRL`location_of_entity_for_lazy_loading`;
  static $keyProps = ['id'];

  cookie: Cookie; // Transient property must be recomputed on rehydration
}
```

Explanation:

```typescript
// Example for retrieving a `Entity`.
const userEntity: UserEntity = UserEntity.$hydrate(someElement, 'user:some_user_id');

/**
 * A transient `Entity` instance.
 *
 * If the application gets hydrated a new instance of `UserEntity` will be created.
 */
userEntity;

/**
 * A transient `Entity` property.
 *
 * It will not be serialized and rehydrated. `UserEntity` will have to recompute it.
 */
userEntity.cookie;

/**
 * An immutable constant `Props` that identifies a `Entity`.
 *
 * `Props` which are parsed from the `EntityKey`.
 */
expect(userEntity.$props).toEqual({ id: 'some_user_id' });

/**
 * The serializable state of a `Entity`.
 *
 * A entity `State` is either rehydrated from DOM/HTML or the `Entity` can communicate with an external resource
 * (e.g. a server) to look up the associated `State` based on the `EntityKey`.
 */
expect(userEntity.$state).toEqual({
  fullName: 'Joe Someone',
  age: 20,
});
```

<<<<<<< HEAD
`Component`s are similar to `Entity`s except they are associated with a specific UI host-element, and a `Component`'s `Props` can change over time.

## DOM Centric

A Qwik application is DOM-centric. All of the information about the application `Component`s, `Entity`s, `Event`s, and entity bindings are stored in the DOM as custom attributes. There is no runtime Qwik framework heap state (with the exception of caches for performance). The result is that a Qwik application can easily be rehydrated because the Qwik framework has no runtime information which needs to be recreated on the client.

Here are some common ways Qwik framework keeps state in DOM/HTML.

- `<some-component decl:template="qrl_to_template">`: The `decl:template` attribute identifies a component boundary. It also points to the location where the template can be found in case of rehydration. `Component`s can be rehydrated and rendered independently of each other.
- `<div ::user="qrl_to_service">`: The `::user` attribute declares a `UserEntity` provider on this element's injector which points to the location where the `Entity` can be lazy loaded from.
- `<div :user:some_user_id='{"fullName": "Joe Someone", "age": 20}'>`: The dehydrated, serialized form of a `UserEntity` with `Props: {id: 'some_user_id'}` and `State: {fullName: "Joe Someone", age: 20}`.
- `<some-component bind:user:some_user_id="$user">`: A entity binding to a `Component`. This tells Qwik that if the `State` of `UserEntity ` with `EntityKey`: `user:some_user_id` changes, the component `<some-component>` will need to be re-rendered.
- # `<some-component on:click="qrl_to_handler">`: The `on:click` attribute notifies Qwik framework that the component is interested in `click` events. The attribute points to the location where the click handler can be lazy-loaded from.
  `Component`s are similar to `Entity`s except they are associated with a specific UI host-element, and `Component`'s `Props` can change over time.

## DOM Centric

A Qwik application is DOM-centric. All of the information about the application `Component`s, `Entity`s, `Event`s, and entity bindings are stored in the DOM as custom attributes. There is no runtime Qwik framework heap state (with the exception of caches for performance). The result is that Qwik application can easily be rehydrated because Qwik framework has no runtime information which needs to be recreated on the client.

Here are some common ways Qwik framework keeps state in DOM/HTML.

- `<some-component decl:template="qrl_to_template">`: The `::` attribute identifies a component boundary. It also points to the location where the template can be found in case of rehydration. `Component`s can be rehydrated and rendered independently of each other.
- `<div ::user="qrl_to_entity">`: The `::user` attribute declares a `UserEntity` provider which points to the location where the `Entity` can be lazy loaded from.
- `<div :user:some_user_id='{"fullName": "Joe Someone", "age": 20}'>`: A serialized form of a `UserEntity` with `Props: {id: 'some_user_id'}` and `State: {fullName: "Joe Someone", age: 20}`.
- `<some-component bind:user:some_user_id="$user">`: A entity binding to a `Component`. This tells Qwik that if the `State` of `UserEntity ` with `Key`: `user:some_user_id` changes, the component `<some-component>` will need to be re-rendered.
- `<some-component on:click="qrl_to_handler">`: The `on:click` attribute notifies Qwik framework that the component is interested in the `click` events. The attribute points to the location where the click handler can be lazy-loaded from.

The benefit of the DOM centric approach is that all of the application state is already serialized in DOM/HTML. The Qwik framework itself has no additional information which it needs to store about the application.

Another important benefit is that Qwik can use `querySelectorAll` to easily determine if there are any bindings for a `Entity` or if there are any listeners for a specific `Event` without having to consult any internal data structures. The DOM is Qwik's framework state.
