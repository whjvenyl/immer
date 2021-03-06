# Immer

[![npm](https://img.shields.io/npm/v/immer.svg)](https://www.npmjs.com/package/immer) [![size](http://img.badgesize.io/https://cdn.jsdelivr.net/npm/immer/dist/immer.umd.js?compression=gzip)](http://img.badgesize.io/https://cdn.jsdelivr.net/npm/immer/dist/immer.umd.js) [![install size](https://packagephobia.now.sh/badge?p=immer)](https://packagephobia.now.sh/result?p=immer) [![Build Status](https://travis-ci.org/mweststrate/immer.svg?branch=master)](https://travis-ci.org/mweststrate/immer) [![Coverage Status](https://coveralls.io/repos/github/mweststrate/immer/badge.svg?branch=master)](https://coveralls.io/github/mweststrate/immer?branch=master) [![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg)](https://github.com/prettier/prettier) [![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.me/michelweststrate)

_Create the next immutable state tree by simply modifying the current tree_

---

* NPM: `npm install immer`
* Yarn: `yarn add immer`
* CDN: Exposed global is `immer`
  * Unpkg: `<script src="https://unpkg.com/immer/dist/immer.umd.js"></script>`
  * JSDelivr: `<script src="https://cdn.jsdelivr.net/npm/immer/dist/immer.umd.js"></script>`

---

* Egghead lesson covering all of immer (7m): [Simplify creating immutable data trees with Immer](https://egghead.io/lessons/redux-simplify-creating-immutable-data-trees-with-immer)
* Introduction blogpost: [Immer: Immutability the easy way](https://medium.com/@mweststrate/introducing-immer-immutability-the-easy-way-9d73d8f71cb3)
* [Talk](https://www.youtube.com/watch?v=-gJbS7YjcSo) + [slides](http://immer.surge.sh/) on Immer at React Finland 2018 by Michel Weststrate

Immer (German for: always) is a tiny package that allows you to work with immutable state in a more convenient way. It is based on the [_copy-on-write_](https://en.wikipedia.org/wiki/Copy-on-write) mechanism.

The basic idea is that you will apply all your changes to a temporarily _draftState_, which is a proxy of the _currentState_. Once all your mutations are completed, Immer will produce the _nextState_ based on the mutations to the draft state. This means that you can interact with your data by simply modifying it, while keeping all the benefits of immutable data.

![immer-hd.png](images/hd/immer.png)

Using Immer is like having a personal assistant; he takes a letter (the current state), and gives you a copy (draft) to jot changes onto. Once you are done, the assistant will take your draft and produce the real immutable, final letter for you (the next state).

A mindful reader might notice that this is quite similar to `withMutations` of ImmutableJS. It is indeed, but generalized and applied to plain, native JavaScript data structures (arrays and objects) without further needing any library.

## API

The Immer package exposes a default function that does all the work.

`produce(currentState, producer: (draftState) => void): nextState`

There is also a curried overload that is explained [below](#currying).

## Example

```javascript
import produce from "immer"

const baseState = [
    {
        todo: "Learn typescript",
        done: true
    },
    {
        todo: "Try immer",
        done: false
    }
]

const nextState = produce(baseState, draftState => {
    draftState.push({todo: "Tweet about it"})
    draftState[1].done = true
})
```

The interesting thing about Immer is that the `baseState` will be untouched, but the `nextState` will reflect all changes made to `draftState`.

```javascript
// the new item is only added to the next state,
// base state is unmodified
expect(baseState.length).toBe(2)
expect(nextState.length).toBe(3)

// same for the changed 'done' prop
expect(baseState[1].done).toBe(false)
expect(nextState[1].done).toBe(true)

// unchanged data is structurally shared
expect(nextState[0]).toBe(baseState[0])
// changed data not (dûh)
expect(nextState[1]).not.toBe(baseState[1])
```

## Benefits

* Immutability with normal JavaScript objects and arrays. No new APIs to learn!
* Strongly typed, no string based paths selectors etc.
* Structural sharing out of the box
* Object freezing out of the box
* Deep updates are a breeze
* Boilerplate reduction. Less noise, more concise code.
* Small: bundled and minified: 2KB.

Read further to see all these benefits explained.

## Reducer Example

Here is a simple example of the difference that Immer could make in practice.

```javascript
// Redux reducer
// Shortened, based on: https://github.com/reactjs/redux/blob/master/examples/shopping-cart/src/reducers/products.js
const byId = (state, action) => {
    switch (action.type) {
        case RECEIVE_PRODUCTS:
            return {
                ...state,
                ...action.products.reduce((obj, product) => {
                    obj[product.id] = product
                    return obj
                }, {})
            }
        default:
            return state
    }
}
```

After using Immer, that simply becomes:

```javascript
import produce from "immer"

const byId = (state, action) =>
    produce(state, draft => {
        switch (action.type) {
            case RECEIVE_PRODUCTS:
                action.products.forEach(product => {
                    draft[product.id] = product
                })
        }
    })
```

Notice that it is not needed to handle the default case, a producer that doesn't do anything will simply return the original state.

Creating Redux reducer is just a sample application of the Immer package. Immer is not just designed to simplify Redux reducers. It can be used in any context where you have an immutable data tree that you want to clone and modify (with structural sharing).

_Note: it might be tempting after using producers for a while, to just place `produce` in your root reducer and then pass the draft to each reducer and work directly over such draft. Don't do that. It kills the point of Redux where each reducer is testable as pure reducer. Immer is best used when applying it to small individual pieces of logic._

## React.setState example

Deep updates in the state of React components can be greatly simplified as well by using immer. Take for example the following onClick handlers (Try in [codesandbox](https://codesandbox.io/s/m4yp57632j)):

```javascript
/**
 * Classic React.setState with a deep merge
 */
onBirthDayClick1 = () => {
    this.setState(prevState => ({
        user: {
            ...prevState.user,
            age: prevState.user.age + 1
        }
    }))
}

/**
 * ...But, since setState accepts functions,
 * we can just create a curried producer and further simplify!
 */
onBirthDayClick2 = () => {
    this.setState(
        produce(draft => {
            draft.user.age += 1
        })
    )
}
```

## Currying

Passing a function as the first argument to `produce` is intended to be used for currying. This means that you get a pre-bound producer that only needs a state to produce the value from. The producer function gets passed in the draft, and any further arguments that were passed to the curried function.

For example:

```javascript
// mapper will be of signature (state, index) => state
const mapper = produce((draft, index) => {
    draft.index = index
})

// example usage
console.dir([{}, {}, {}].map(mapper))
//[{index: 0}, {index: 1}, {index: 2}])
```

This mechanism can also nicely be leveraged to further simplify our example reducer:

```javascript
import produce from 'immer'

const byId = produce((draft, action) => {
  switch (action.type) {
    case RECEIVE_PRODUCTS:
      action.products.forEach(product => {
        draft[product.id] = product
      })
      return
    })
  }
})
```

Note that `state` is now factored out (the created reducer will accept a state, and invoke the bound producer with it).

If you want to initialize an uninitialized state using this construction, you can do so by passing the initial state as second argument to `produce`:

```javascript
import produce from "immer"

const byId = produce(
    (draft, action) => {
        switch (action.type) {
            case RECEIVE_PRODUCTS:
                action.products.forEach(product => {
                    draft[product.id] = product
                })
                return
        }
    },
    {
        1: {id: 1, name: "product-1"}
    }
)
```

## Auto freezing

Immer automatically freezes any state trees that are modified using `produce`. This protects against accidental modifications of the state tree outside of a producer. This comes with a performance impact, so it is recommended to disable this option in production. It is by default enabled. By default it is turned on during local development, and turned off in production. Use `setAutoFreeze(true / false)` to explicitly turn this feature on or off.

## Returning data from producers

It is not needed to return anything from a producer, as Immer will return the (finalized) version of the `draft` anyway. However, it is allowed to just `return draft`.

It is also allowed to return arbitrarily other data from the producer function. But _only_ if you didn't modify the draft. This can be useful to produce an entirely new state. Some examples:

```javascript
const userReducer = produce((draft, action) => {
    switch (action.type) {
        case "renameUser":
            // OK: we modify the current state
            draft.users[action.payload.id].name = action.payload.name
            return draft // same as just 'return'
        case "loadUsers":
            // OK: we return an entirely new state
            return action.payload
        case "adduser-1":
            // NOT OK: This doesn't do change the draft nor return a new state!
            // It doesn't modify the draft (it just redeclares it)
            // In fact, this just doesn't do anything at all
            draft = {users: [...draft.users, action.payload]}
            return
        case "adduser-2":
            // NOT OK: modifying draft *and* returning a new state
            draft.userCount += 1
            return {users: [...draft.users, action.payload]}
        case "adduser-3":
            // OK: returning a new state. But, unnecessary complex and expensive
            return {
                userCount: draft.userCount + 1,
                users: [...draft.users, action.payload]
            }
        case "adduser-4":
            // OK: the immer way
            draft.userCount += 1
            draft.users.push(action.payload)
            return
    }
})
```

## Using `this`

The recipe will be always invoked with the `draft` as `this` context.

This means that the following constructions are also valid:

```javascript
const base = {counter: 0}

const next = produce(base, function() {
    this.counter++
})
console.log(next.counter) // 1

// OR
const increment = produce(function() {
    this.counter++
})
console.log(increment(base).counter) // 1
```

## TypeScript or Flow

The Immer package ships with type definitions inside the package, which should be picked up by TypeScript and Flow out of the box and without further configuration.

The TypeScript typings automatically remove `readonly` modifiers from your draft types and return a value that matches your original type. See this practical example:

```ts
import produce from 'immer';

interface State {
  readonly x: number;
}

// `x` cannot be modified here
const state: State = {
  x: 0;
};

const newState = produce<State>(draft => {
  // `x` can be modified here
  draft.x++;
});

// `newState.x` cannot be modified here
```

This ensures that the only place you can modify your state is in your produce callbacks. It even works recursively and with `ReadonlyArray`s!

## Immer on older JavaScript environments?

By default `produce` tries to use proxies for optimal performance. However, on older JavaScript engines `Proxy` is not available. For example, when running Microsoft Internet Explorer or React Native on Android. In such cases Immer will fallback to an ES5 compatible implementation which works identical, but is a bit slower.

## Pitfalls

1. Don't redefine draft like, `draft = myCoolNewState`. Instead, either modify the `draft` or return a new state. See [Returning data from producers](#returning-data-from-producers).
1. Currently, Immer only supports plain objects and arrays. PRs are welcome for more language built-in types like `Map` and `Set`.
1. Immer only processes native arrays and plain objects (with a prototype of `null` or `Object`). Any other type of value will be treated verbatim! So if you modify a `Map` or `Buffer` (or whatever complex object from the draft state), the changes will be persisted. But, both in your new and old state! So, in such cases, make sure to always produce fresh instances if you want to keep your state truly immutable.
1. For example, working with `Date` objects is no problem, just make sure you never modify them (by using methods like `setYear` on an existing instance). Instead, always create fresh `Date` instances. Which is probably what you were unconsciously doing already.
1. Since Immer uses proxies, reading huge amounts of data from state comes with an overhead (especially in the ES5 implementation). If this ever becomes an issue (measure before you optimize!), do the current state analysis before entering the producer function or read from the `currentState` rather than the `draftState`
1. Some debuggers (at least Node 6 is known) have trouble debugging when Proxies are in play. Node 8 is known to work correctly.

## Cool things built with immer

* [react-copy-write](https://github.com/aweary/react-copy-write) _Immutable state with a mutable API_
* [redux-starter-kit](https://github.com/markerikson/redux-starter-kit) _A simple set of tools to make using Redux easier_
* [immer based handleActions](https://gist.github.com/kitze/fb65f527803a93fb2803ce79a792fff8) _Boilerplate free actions for Redux_
* [redux-box](https://github.com/anish000kumar/redux-box) _Modular and easy-to-grasp redux based state management, with least boilerplate_
* [quick-redux](https://github.com/jeffreyyoung/quick-redux) _tools to make redux developement quicker and easier_
* [bey](https://github.com/jamiebuilds/bey) _Simple immutable state for React using Immer_
* ... and [many more](https://www.npmjs.com/browse/depended/immer)

## How does Immer work?

Read the (second part of the) [introduction blog](https://medium.com/@mweststrate/introducing-immer-immutability-the-easy-way-9d73d8f71cb3).

## Example patterns.

_For those who have to go back to thinking in object updates :-)_

```javascript
import produce from "immer"

// object mutations
const todosObj = {
    id1: {done: false, body: "Take out the trash"},
    id2: {done: false, body: "Check Email"}
}

// add
const addedTodosObj = produce(todosObj, draft => {
    draft["id3"] = {done: false, body: "Buy bananas"}
})

// delete
const deletedTodosObj = produce(todosObj, draft => {
    delete draft["id1"]
})

// update
const updatedTodosObj = produce(todosObj, draft => {
    draft["id1"].done = true
})

// array mutations
const todosArray = [
    {id: "id1", done: false, body: "Take out the trash"},
    {id: "id2", done: false, body: "Check Email"}
]

// add
const addedTodosArray = produce(todosArray, draft => {
    draft.push({id: "id3", done: false, body: "Buy bananas"})
})

// delete
const deletedTodosArray = produce(todosArray, draft => {
    draft.splice(draft.findIndex(todo => todo.id === "id1"), 1)
    // or (slower):
    // return draft.filter(todo => todo.id !== "id1")
})

// update
const updatedTodosArray = produce(todosArray, draft => {
    draft[draft.findIndex(todo => todo.id === "id1")].done = true
})
```

## Performance

Here is a [simple benchmark](__performance_tests__/todo.js) on the performance of Immer. This test takes 100.000 todo items, and updates 10.000 of them. _Freeze_ indicates that the state tree has been frozen after producing it. This is a _development_ best practice, as it prevents developers from accidentally modifying the state tree.

These tests were executed on Node 8.4.0. Use `yarn test:perf` to reproduce them locally.

![performance.png](images/performance.png)

Some observations:

* From `immer` perspective, this benchmark is a _worst case_ scenario, because the root collection it has to proxy is really large relatively to the rest of the data set.
* The _mutate_, and _deepclone, mutate_ benchmarks establish a baseline on how expensive changing the data is, without immutability (or structural sharing in the deep clone case).
* The _reducer_ and _naive reducer_ are implemented in typical Redux style reducers. The "smart" implementation slices the collection first, and then maps and freezes only the relevant todos. The "naive" implementation just maps over and processes the entire collection.
* Immer with proxies is roughly speaking twice as slow as a hand written reducer. This is in practice negligible.
* Immer is roughly as fast as ImmutableJS. However, the _immutableJS + toJS_ makes clear the cost that often needs to be paid later; converting the immutableJS objects back to plain objects, to be able to pass them to components, over the network etc... (And there is also the upfront cost of converting data received from e.g. the server to immutable JS)
* The ES5 implementation of Immer is significantly slower. For most reducers this won't matter, but reducers that process large amounts of data might benefit from not (or only partially) using an Immer producer. Luckily, Immer is fully opt-in.
* The peaks in the _frozen_ versions of _just mutate_, _deepclone_ and _naive reducer_ come from the fact that they recursively freeze the full state tree, while the other test cases only freeze the modified parts of the tree.

## FAQ

_(for those who skimmed the above instead of actually reading)_

**Q: Does Immer use structural sharing? So that my selectors can be memoized and such?**

A: Yes

**Q: Does Immer support deep updates?**

A: Yes

**Q: I can't rely on Proxies being present on my target environments. Can I use Immer?**

A: Yes

**Q: Can I typecheck my data structures when using Immer?**

A: Yes

**Q: Can I store `Date` objects, functions etc in my state tree when using Immer?**

A: Yes

**Q: Is it fast?**

A: Yes

**Q: Idea! Can Immer freeze the state for me?**

A: Yes

## Credits

Special thanks goes to @Mendix, which supports it's employees to experiment completely freely two full days a month, which formed the kick-start for this project.

## Donations

A significant part of my OSS work is unpaid. So [donations](https://mobx.js.org/donate.html) are greatly appreciated :)
