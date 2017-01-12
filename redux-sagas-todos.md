# redux-saga Todos Tutorial

**NOTE: Some examples are outdated. The react-redux-starter-kit has changed since this was first written.**

In this tutorial we're going to implement [redux-saga](http://yelouafi.github.io/redux-saga/) in the same [Todos app from the previous document](./react-redux-starter-kit-todos.md).

The power of sagas becomes aparent the moment you try to hook redux to an API, which we'll be doing in the next step. In this tutorial we won't be writing sagas to fetch data from an API. This time around we're going to get sagas integrated into our app so that we can focus on the API portion of it later. So, why should you read this? Because setting up redux-saga with react-redux-starter-kit isn't obvious and it's important to cover the fundamentals of sagas in depth first. If you try to use redux-saga for the first time it can be intimidating because most tutorials jump right into [using redux-saga to interact with an API](https://github.com/yelouafi/redux-saga#usage-example). We're going to work up to it slowly.

If you want to get into redux-saga they have a [great beginner's tutorial](http://yelouafi.github.io/redux-saga/docs/introduction/BeginnerTutorial.html). You may also want to read some of their [saga background links](http://yelouafi.github.io/redux-saga/docs/introduction/SagaBackground.html) which cover where the core concepts came from.

We'll cover it more later but redux-saga makes extensive use of [generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*). You can read [in depth about generators](http://www.2ality.com/2015/03/es6-generators.html) if you'd like but [the basics](https://davidwalsh.name/es6-generators) become clear quite quickly.

### How do generators work?
Because generators are unfamiliar to most JavaScript developers, much of the documentation about sagas will seem dense and confusing. Don't get discouraged. Generators are really easy.

Test this example out in the online [JavaScript REPL](https://repl.it/CQPk/1)

```js
// A generator function has a star
function * test () {
  // you mark the "steps" with yield
  yield 'hello'
  yield 'goodbye'
  return 'done' // <-- you don't normally return something this way, but you can
}

// calling a generator function returns a generator object
const task = test()

// you "walk" the generator using next
task.next() // --> { value: 'hello', done: false }
task.next() // --> { value: 'goodbye', done: false }
task.next() // --> { value: 'done', done: true }
```

Get it? You "walk" a generator function in "steps". The function "pauses" after each step.

1. Call a generator to get a [Generator object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator).
2. Call [`next()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/next) to step through each `yield`
3. The [`yield`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield) keyword means "return and pause", it's why you don't normally see a `return` in generator. If you do add in a `return` it will be used as the `value` on the very last `next()` call.
4. The generator stays paused until the `next()` function is called again.
5. You should read about [Iterators and Generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators)

The important part of generators is that you must walk your generators manually. While walking a generator seems like an annoying task, **redux-saga walks your generators for you. That's it's best feature!**

### What is redux-saga?
There is an excellent write up on [why redux-saga is great](http://riadbenguella.com/from-actions-creators-to-sagas-redux-upgraded/). You probably want to read about sagas to get [an authoratative definition](https://msdn.microsoft.com/en-us/library/jj591569.aspx) -- don't bother. In the simplest terms  redux-sagas is a task runner -- it literally runs whatever function you give it. Of course redux-saga is specially designed for running generators. We'll see later on that the main "software" that comes in the redux-saga package is the sagaMiddleware. Read some of these [saga resources](http://yelouafi.github.io/redux-saga/docs/ExternalResources.html) for more information.

You use it like this: [`sagaMiddleware.run(saga)`](http://yelouafi.github.io/redux-saga/docs/api/index.html#middlewarerunsaga-args). Remember that.

A `saga` is just a function. The saga middleware runs it after the next action is dispatched in your redux app. Your saga function recieves `action` as the first argument. Redux-saga works a little differently than other middleware because you have to manage when your sagas are run. In practice this is no more difficult than managing reducers.

Usually you configure your sagas to "stay alive" and keep listening for more actions, but you don't have to do it that way.

Here's an example of a saga that's only configured to run once. We'll see later that the whole point of your saga is to yield effects. When you use the `sagaMiddleware` your saga will be run when the next action is dispatched. It's important to note that usually your saga will only be run once. However it's common to use a watcher saga to keep responding to future actions. Although you don't have to use a watcher, it's the most common way you'll use the saga middelware. We'll see a little further down what watching for specific actions looks like.

```js
// a saga is just a generator function that recieves an action
function * mySaga (action) {
  // usually you'd yield an effect instead of true
  // you can yield effects to manage how your saga is walked
  yield true
}

// you run your saga with the middleware
// your saga will be walked when the next action is dispatched
sagaMiddleware.run(mySaga)
```

*Note:* Normally you'd yield an effect like [`put()`](http://yelouafi.github.io/redux-saga/docs/api/index.html#putaction) instead of `true`.

In a normal middleware like redux-thunk the asynchronous functionality gets baked directly into your action. In redux-saga it's different. When you watch a saga with `takeEvery`, **the *saga* is the *target* of an action**, more like a reducer. The `sagaMiddleware` is the bridge between dispatched actions and your saga. If you find yourself working with the sagaMiddleware all the time you're probably doing something pretty advanced. Usually you'll want to run your sagas as part of your route. We'll see that in practice later on.

What's important is that **your saga is just a function... a *generator* function.**

Here's an example of using the `takeEvery()` function from redux-saga to create a watcher for our saga. You use takeEvery to create a watcher for your saga. This ensures that our saga is run when a specific action is dispatched. It also ensures that your saga will keep running to capture future actions. You should read up on [how `takeEvery()` works](http://yelouafi.github.io/redux-saga/docs/api/index.html#takeeverypattern-saga-args).

```js
function * mySaga (action) {
  yield true
}

// you can use takeEvery to watch an actionType
// and run your saga everytime it is dispatched
const createWatcher = (actionType, saga) => {
  return function * () {
    yield * takeEvery(actionType, saga)
  }
}

// you run your watcher with the middleware
sagaMiddleware.run(createWatcher(ACTION_NAME, mySaga))
```

In a similar way to how a reducer is just a *function* that responds to an action, **a saga is just a *generator* that responds to an action.** Redux-saga allows you to *subscribe* to actions. Of course the reason you want to subscribe to an action is to fire more actions! A saga is a great way to do a bunch of things in sequence, even if some of the things require waiting.

The classic example is [using redux-saga to make a `fetch()` request](http://yelouafi.github.io/redux-saga/docs/basics/UsingSagaHelpers.html). Because an [AJAX request](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) is asynchronous you can't handle it with the normal `dispatch->reducer` workflow. With asynchronous actions you have to use middleware to manage the flow; redux is strictly synchronous. An async flow looks more like like `dispatch->middleware( saga->dispatch->reducer )`. After an initial action is dispatched, your saga needs to dispatch additional actions to update the application state as the promise completes. In [the classic redux async example](http://redux.js.org/docs/advanced/ExampleRedditAPI.html) this means using thunks to dispatch a series of actions. When you use thunks everything happens in your action and the resulting code can start to look messy. Redux-saga makes this much easier.

```js
// we should return a promise when we're trying to fetch something
const fetchSomething = () => {
  const url = ''
  return Promise ((resolve, reject) => {
    try {
      fetch(url).then(response => resolve(response))
    } catch(error) {
      reject(error)
    }
  })
}

// and we use a saga to fetch something when an action is dispatched
function * mySaga (action) {

  // redux-saga manages promises for you
  // your saga is resumed after the promise is resolved
  // you can capture the response of a promise
  const reponse = yield call(fetchSomething)

  // ... do something with the response
}

// you can use takeEvery to watch an actionType
// and run your saga everytime it is dispatched
const createWatcher = (actionType, saga) => {
  return function * () {
    yield * takeEvery(actionType, saga)
  }
}

sagaMiddleware.run(createWatcher(FETCH_SOMETHING, mySaga))
```

When you create a watcher, **the saga is the target of an action, not the action itself.** Sagas usually dispatch additional actions after waiting for an asynchronous response.

### Different ways to manage async actions in JavaScript
To manage asynchronous actions, redux-saga utilizes generator functions. Generators were *specifically* designed to manage asynchronous actions. If you have been trying to get used to Promises, then you'll get the root concepts right away. If you're used to callbacks you'll quickly see why this is better. A generator looks a lot like a [promise chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then) or a [callback hell](http://callbackhell.com/).

You might enjoying reading [an excellent overview of how JavaScript manages async](https://medium.com/@rdsubhas/es6-from-callbacks-to-promises-to-generators-87f1c0cd8f2e). Below is an overview of the typical ways you can manage asyncronous actions in JavaScript.

##### A promise chain
In a promise chain you can keep adding `then()` functions to a promise. It ensures that your additional actions execute only after the initial promise has resolved. Check out a more [complete example in the JavaScript REPL](https://repl.it/CR47/9).

```js
// a promise chain
fetchSomething()
  .then(result => firstThing(result))
  .then(result => secondThing(result))
```

##### A callback hell
You should read about [callback hell](http://callbackhell.com/) and also how [reactive programming is better](http://stackoverflow.com/questions/25098066/what-is-callback-hell-and-how-and-why-rx-solves-it). Check out a more [complete example in the JavaScript REPL](https://repl.it/CR47/8).

```js
// a callback hell
fetchSomething(result => {
  firstThing(result, result => {
    secondThing(result)
  })
})
```

##### A generator function
Generators allow functions to be paused. It's basically the same thing as a callback hell or a promise chain but it gives the user much more control. You have to walk your generators (see below). Check out a more [complete example in the JavaScript REPL](https://repl.it/CR47/11).

```js
// a generator function
function * doThingsWithSomething () {
  const result = yield fetchSomething()
  yield firstThing(result)
  yield secondThing(result)
}

// you have to walk your generator object
const task = doThingsWithSomething()
```


##### Walking a generator function
Generators give you a lot of control from the outside. It's subtle in the above example but the value captured in `result = yield fetchSomething()` is actually passed in from the outside using the [`next()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/next) function. Internally redux-saga takes full advantage of this, except it passes in the result from the previous effect. If this doesn't make sense don't worry -- redux-saga does this for you!

```js
// you can walk a generator
// (redux-saga does this for you)
function walkTask(task, result = {}) {
  // iterate over the generator object until it is done
  while (!result.done) {
    result = task.next(result.value) // <-- call next() with the previously yielded value

    // redux-saga waits for promises to finish
    if (typeof result.value.then === 'function') {

      // pause walking by returning a promise
      return result.value.then( (response) => {
        result.value = response // <-- override the value with the promise response
        walkTask(task, result) // <-- resume walking
      })
    }
  }

  return result.value
}

walkTask(task)
```

*Note:* The `walkTask()` function above is just a toy example. Redux-saga actually analyzes the effect you yield and does some magic. However, once redux-saga is done with its internal magic it simply calls `next()` on your saga with the result of the previous effect.

### Managing async actions with a saga
Redux-saga allows you to chain actions in an easy-to-follow way. As a core requirement **redux-saga expects you to yield an effect**. Effects help the `sagaMiddleware` walk your saga. Thankfully redux-saga comes with helper functions that make it seamless to [create effects](http://yelouafi.github.io/redux-saga/docs/api/index.html#effect-creators). In practice you'll use [`put()`](http://yelouafi.github.io/redux-saga/docs/api/index.html#putaction) -- essentially an alias for `dispatch()` -- to dispatch actions. And you'll use [`call()`](http://yelouafi.github.io/redux-saga/docs/api/index.html#callfn-args) to call a function that does something asynchronously, either returning a generator or a promise.

[Redux-saga has a glossary](http://yelouafi.github.io/redux-saga/docs/Glossary.html) that comes in handy.

##### A saga
Here's the async example from above rewritten as a saga:

```js
// a saga
function * doThingsWithSomething () {
  const result = yield call(fetchSomething) // <-- fetchSomething returns a promise
  yield put(firstThing(result)) // <-- firstThing is an action, result is the payload
  yield put(secondThing(result))
}
```

Here's some important notes about sagas:

1. Sagas should not update the store directly, that's what a *reducer* is for.
2. Sagas are for fielding actions *before* they get to reducers, that's why redux-saga provides middleware.
3. Redux-saga uses weird terminology like `put` instead of `dispatch` and `takeEvery` instead of `handle`.
4. Redux-saga manages your generators for you, that's why you normally don't have to call [`next()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/next) on your sagas. Your saga should `yield` an [effect](http://yelouafi.github.io/redux-saga/docs/basics/DeclarativeEffects.html) in order to tell redux-saga how to run your generator.

**A saga is a generator that yields effects.**

# Install redux-saga
Let's install [redux-saga](https://github.com/yelouafi/redux-saga):

```bash
npm install redux-saga --save

# you may want to install other common packages for working with modules
npm install redux-actions reselect --save
```

## Apply Saga Middleware
Redux-saga is middleware. When you're working with Redux you need middleware to handle asynchronous actions, these are called side effects. Redux-thunk works well for side effects but sagas make it [much easier to chain your effects](http://stackoverflow.com/questions/32925837/how-to-handle-complex-side-effects-in-redux). The biggest advantage of redux-saga is that it uses native generator functions that are perfectly suited to the types of complex chained actions that you need for making asynchronous requests.

*Note:* If you use redux-saga for your asynchronous actions you shouldn't need [redux-thunk](https://github.com/gaearon/redux-thunk), the middleware that comes with react-redux-starter kit. However we'll be keeping redux-thunk around for now, it doesn't hurt anything to use more than one type of middleware on a project.

### Add redux-saga to createStore
We want to reuse the clever way that react-redux-starter-kit loads the reducers for a route using `injectReducers()`. You can read about it in detail [in the previous note](react-redux-starter-kit-todos.md#route-srcroutestodosindexjs). At a high level, react-redux-starter-kit provides an example of using Webpack to break routes into dynamic chunks. Part of that functionality involves loading the reducers for that route. This is helpful when you're using the [fractal project structure](https://github.com/davezuko/react-redux-starter-kit/wiki/Fractal-Project-Structure).

Thankfully redux-saga has [support for dynamically loading sagas](https://github.com/yelouafi/redux-saga/releases/tag/v0.1.0) in a similar fashion to the `injectReducer()` method that comes with react-redux-starter-kit. You may like to read about [how to dynamically load reducers](https://github.com/reactjs/redux/issues/37).

*Note:* You may want to read about [the need for dynamically loading sagas](https://github.com/yelouafi/redux-saga/issues/76).

We need to import our rootSaga and the sagaMiddleware from our `sagas.js` file (we'll make that file next). We also need to run our middleware with the root saga before returning the store. Redux-saga requires you to "run" your sagas. It's not important how it works but you can't skip this step.

The basics look like this:

We need to add some code to the `createStore.js` to integrate the sagaMiddleware.

##### `src/store/createStore.js`
(compare to the [starter-kit version](https://github.com/davezuko/react-redux-starter-kit/blob/master/src/store/createStore.js))

```js
// ... snippet from src/store/createStore.js

// we're already importing our reducers
import makeRootReducer from './reducers'

// we also need to import our sagas
// (we'll create the sagas.js file below)
import sagaMiddleware, { rootSaga } from './sagas'

// this is the createStore function
export default (initialState = {}, history) => {
  // ======================================================
  // Middleware Configuration
  // ======================================================
  const middleware = [thunk, routerMiddleware(history), sagaMiddleware] // <-- add it to the list

  // ...

  // we need to run our sagas! don't forget!
  sagaMiddleware.run(rootSaga)

  return store
}
```

##### Tip: Don't run an empty root saga
If you are not running any sagas app-wide then you can shorten the code above to simply apply the middleware. You might do this if your `src/store/sagas.js` file (see below) returns an empty root saga.

##### `src/store/createStore.js` (without app-wide sagas)
This shows the same alterations to `createStore.js` above with the `rootSaga` commented out. In many cases you don't need a rootSaga at the application level. It all depends on your app.

```js
// ... alternative snippet from src/store/createStore.js

import makeRootReducer from './reducers'
import sagaMiddleware from './sagas' // <-- don't import the root saga if you don't need to

export default (initialState = {}, history) => {
  const middleware = [thunk, routerMiddleware(history), sagaMiddleware]

  // ...

  // we purposely don't run our root saga
  // because we only run sagas from our routes (common)
  // sagaMiddleware.run(rootSaga)

  return store
}
```


### Add root sagas
We need to create our `src/store/sagas.js` file to provide similar functionality to the `src/store/reducers.js` file. We need to add an `injectSaga()` function that works similarly to `injectReducer()`.

Within redux-saga, the [`sagaMiddleware.run(saga)`](http://yelouafi.github.io/redux-saga/docs/api/index.html#middlewarerunsaga-args) function returns a [`task`](http://yelouafi.github.io/redux-saga/docs/api/index.html#task-descriptor). Redux-saga creates a new task every time it is called. It doens't check if a task for the same saga is already running. This is important because redux-saga only runs your tasks you have to manage your tasks yourself. In practice this is easy but if you're not careful you might accidentally start the same saga twice. If you find yourself with double execution bugs, then you're probably running the same saga more than once.

Calling `injectSaga()` twice with the same arguments prevents double execution.

We also need a `cancelTask(name)` function for canceling our named tasks. This makes it possible to manage sagas dynamically from our routes. We run our sagas when we enter a route; cancel them when we leave a route.

We need to create a `sagas.js` file.

```bash
touch src/store/sagas.js
```

##### `src/store/sagas.js`
We'll dig into this step-by-step below. We create a `sagas.js` file to operate similarly to [`reducers.js`](https://github.com/davezuko/react-redux-starter-kit/blob/master/src/store/reducers.js). Sagas and reducers do different things but they are both potential targets for actions so it's sensible to configure them similarly.

```js
import createSagaMiddleware, { takeEvery } from 'redux-saga'

// sync sagas
// import { helloSaga, watchIncrementAsync } from '../redux/modules/example'

export const sagaMiddleware = createSagaMiddleware()
export const runSaga = (saga) => sagaMiddleware.run(saga)

// use injectSaga() to import from a route
const tasks = {}
export const injectSaga = ({ name, saga }) => {
  let { task, prevSaga } = tasks[name] || {}

  if (task && prevSaga !== saga) {
    cancelTask(name)
    task = undefined
  }

  if (!task || !task.isRunning()) {
    tasks[name] = {
      task: sagaMiddleware.run(saga),
      prevSaga: saga
    }
  }
}

export const cancelTask = (name) => {
  const { task } = tasks[name]
  tasks[name] = undefined

  if (task) {
    task.cancel()
  }
}

export const createWatcher = (actionType, saga) => {
  return function * () {
    yield * takeEvery(actionType, saga)
  }
}

export const watchActions = (sagas) => {
  const watchers = Object.keys(sagas)
    .map((type) => createWatcher(type, sagas[type])())
  return function * rootSaga () {
    yield watchers
  }
}

export function * rootSaga () {
  yield [
    // Add sync sagas here
  ]
}

export default sagaMiddleware
```

Let's dig into this step-by-step.

#### You can import and run app-wide sagas
We're treating our sagas similarly to reducers. So we'll first look at how react-redux-starter-kit is managing reducers. The [`src/store/reducers.js`](https://github.com/davezuko/react-redux-starter-kit/blob/master/src/store/reducers.js) file is where you'd import a reducer that all of your app would use. Otherwise you'd import reducers within your route using `indectReducer()`. If you look in the `reducers.js` file you'll see that reducers used by your whole app are called "sync reducers." They are loaded synchronously as the app loads. For instance, the `router` reducer from react-router-redux is loaded as a sync reducer.

"Async reducers" are loaded in a route, asynchronously, using webpack. We're going to copy that format and load our "sync sagas" in the `src/store/sagas.js` file and our "async sagas" in our route using `injectSaga()`. You might refer to these types of reducers and sagas as "app sagas" because they're loaded at the app level. As opposed to "route" sagas and reducers which are loaded at the route level.

Here we're pretending that we have some sagas available that we'd like to import app-wide. We'll see later that it's more common to run your sagas when you enter a route. (You can see examples of these sagas in the [beginners tutorial](http://yelouafi.github.io/redux-saga/docs/introduction/BeginnerTutorial.html).)

```js
// ... snippet from src/store/sagas.js

// sync sagas
import { helloSaga, watchIncrementAsync } from '../redux/modules/example'

// ...

export default function * rootSaga () {
  yield [
    // Add sync sagas here
    helloSaga(),
    watchIncrementAsync()
  ]
}
```

#### Export the sagaMiddleware
This is where we create the middleware we used in `src/store/createStore.js`. We create an `injectSaga()` function so that we don't have to muck with the middleware directly outside of `createStore.js`. In order for a saga to respond to redux actions it needs to be "run" by the saga middleware.

```js
// ... middleware snippet from src/store/sagas.js

// this gets used in applyMiddleware()
// we also use it inside of injectSaga() and runSaga()
export const sagaMiddleware = createSagaMiddleware()

// if we just want to run a single saga, we can use this
// (usually when you're doing something clever)
export const runSaga = (saga) => sagaMiddleware.run(saga)

// ...
```

#### Inject some sagas
Here we're bending some of the core terminology of redux-saga to be more in line with what we see elsewhere in react-redux-starter-kit. At it's core, `injectSaga()` is doing the same thing as `sagaMiddleware.run(saga)`. Remember that? This is the same thing but better. Usually you need to launch a "watcher" saga when you enter a route and kill it when you leave a route (more on this later). We'll see clearly how you use this in the next step so you don't have to understand this fully yet. You only need this when you're setting up a route.

```js
// ... injectSaga() snippet from src/store/sagas.js

// run a named saga from a route
export const injectSaga = ({ name, saga }) => {
  let { task, prevSaga } = tasks[name] || {} // <-- find by name

  if (task && prevSaga !== saga) {
    cancelTask(name) // <-- only happens when the name is the same but the saga is different
    task = undefined
  }

  // the name prevents us from running the same saga twice
  if (!task || !task.isRunning()) {
    tasks[name] = {

      // running a saga returns a task
      task: sagaMiddleware.run(saga),
      prevSaga: saga
    }
  }
}
```

#### Watching for actions
When you're building a module you'll want to register your saga to run when certain actions are dispatched. This is very similar to working with a reducer, where you want your reducer to handle certain actions. For reducers we're using [`handleActions()`](https://github.com/acdlite/redux-actions#handleactionsreducermap-defaultstate) from redux-actions. In order to get our sagas to watch for actions we're going to be doing something similar. Under the hood the `watchActions()` function mimics the redux-saga textbook example of creating a watcher saga.

(You can see examples of creating a watcher saga and exporting a rootSaga in the [beginners tutorial](http://yelouafi.github.io/redux-saga/docs/introduction/BeginnerTutorial.html).)

```js
// ... example of using watchActions() inside a module

// ...
import { watchActions } from '../store/sagas.js'

// constants
export const MY_ACTION_TYPE = 'MY_ACTION_TYPE'
export const ANOTHER_ACTION_TYPE = 'ANOTHER_ACTION_TYPE'

// sagas
function * mySaga (action) {
  yield true
}

function * anotherSaga (action) {
  yield true
}

export const rootSaga = watchActions({
  [MY_ACTION_TYPE]: mySaga,
  [ANOTHER_ACTION_TYPE]: anotherSaga
})
```

## Dynamically load a saga from a route
We need to dynamically load our saga in our route. This is nearly identical to the [previous example for our todo route](./react-redux-starter-kit-todos.md#route-srcroutestodosindexjs). We're going to be making changes to that file to make it support our sagas. This will be pretty standard on any route you create that needs to implement sagas.

We'll be importing saga from our modules. We'll see later what modules look like with sagas in them.

First we need to alter the todos route to run our sagas.

##### `src/routes/Todos/index.js`
(compare to [previous `src/routes/Todos/index.js`](./react-redux-starter-kit-todos.md#route-srcroutestodosindexjs))

```js
import { injectReducer } from '../../store/reducers'
import { injectSaga } from '../../store/sagas'

export default (store) => ({
  path: 'todos',
  getComponent (nextState, cb) {
    require.ensure([], (require) => {
      const TodosView = require('./components/TodosView').default
      const { default: reducer, rootSaga: saga } = require('./modules/todos')

      injectReducer(store, { key: 'todosApp', reducer })
      injectSaga({ name: 'todosApp', saga })

      cb(null, TodosView)
    }, 'todos')
  }
})
```

#### How this works
Because we're using Webpack's `require.ensure`, we have to use `require` instead of importing the module directly. Check out [how to rename a variable with destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#Assigning_to_new_variable_names). We'll see later how we construct a typical module. It's best to follow the convention of exporting your reducer by default and exporting your modules sagas as `rootSaga`. We'll see in a second what that looks like, but it's basically what we do in the `src/store/sagas.js`.

```js
// ... snippet from src/routes/Todos/index.js

// we use require to, umm, import our module
// we pull out the default reducer
// and we grab the root saga
const { default: reducer, rootSaga: saga } = require('./modules/todos')

// we inject the reducer as normal
injectReducer(store, { key: 'todosApp', reducer })

// we also run our root saga
injectSaga({ name: 'todosApp', saga })

// ...
```

## About Modules
Before we try to implement a saga in our sample app it's important to go over what a generic module looks like. A module is the redux version of a "model" from a traditional MVC app. Of course it's not *exactly* a model but you can see that all of the main pieces are there. Typically a model has getters and setters. In practice you will probably be organizing your module to look similar to something like an [Ember Data Model](http://emberjs.com/api/data/classes/DS.Model.html).

If you're following my anology, you use a selector in place of a getter. You should use selectors in the `mapStateToProps` function of a container. A selector reads a value from the store. Technically you can store data in redux however you want and there are numerous ways to read data back. Usually you create a selector as a simple function like `const getAllTodos = (state) => state.todosApp.todos.data`.

There are some selectors that you want to memoize because they are expensive. We use [reselect](https://github.com/reactjs/reselect) for this. Regardless of how you structure your modules, using reselect to read from the store is highly recommended. The [documentation provided](https://github.com/reactjs/reselect) is top-notch. More on this below.

If a selector is a getter, then what is a setter? The short answer is "reducers" but that's not the whole story. Typically you don't call a reducer directly, instead reducers handle actions as they are dispatched. You should dispatch actions from the `mapDispatchToProps` function in a container. In redux the simple act of updating an entity is turned into a series of actions and sagas until it eventually reaches a reducer and updates the application state.

When you use something like Ember Data it is very easy to get and set a value on a model. But Ember Data itself does a tremendous amount of work to manage all of the underlying side effects without you needing to worry. In redux there's no magic going on and you have to manage those side effects yourself. That makes it slightly harder to get going but actually results in better performing code and completely removes framework-fighting (bending over backwards to get the framework to do what you want).

1. Selectors are for reading data from the store. We use [reselect](https://github.com/reactjs/reselect).
2. Actions are the first step in writing to the store. We use [redux-actions](https://github.com/acdlite/redux-actions).
3. Sagas are for fielding actions that require async operations. We use [redux-saga](https://github.com/yelouafi/redux-saga).
4. Reducers are for writing to the store. We use [redux-actions](https://github.com/acdlite/redux-actions) for this too.
5. Selectors and Actions form the interface to your module. Anyone using your module in their code will import the selectors for use in `mapStateToProps` and the actions for use in `mapDispatchToProps`.

##### Empty module
Here's an empty module. We'll be filling this in. We've already done this in the [previous todos app note](./react-redux-starter-kit-todos.md#todos-module).

```js
import { combineReducers } from 'redux'
import { watchActions } from '../store/sagas'

// selectors

// constants

// action creators

// sagas

// combine sagas
export const rootSaga = watchActions({
  // combine all of your module's sagas
})

// reducers

// combine reducers
export default combineReducers({
  // combine all of your module's reducers
})
```

##### Generic example module
Here's a generic module that makes use of sagas. It's ok to skim this. We'll be rewriting this below for our todo example we've been working on.

```js
// generic module

import { combineReducers } from 'redux'
import { createAction, handleActions } from 'redux-actions'
import { createSelector } from 'reselect'
import { takeEvery, delay } from 'redux-saga'
import { put } from 'redux-saga/effects'
import { createWatcher } from '../store/sagas'

// selectors
export const getResult = (state) => state.myReducer.result
export const getPending = (state) => state.myReducer.pending

export const getFinalResult = createSelector(
  [ getResult, getPending ],
  (result, pending) => !!pending ? result : undefined
)

// constants
export const MY_ACTION = 'MY_ACTION'
export const MY_ASYNC_ACTION = 'MY_ASYNC_ACTION'
export const ANOTHER_ASYNC_ACTION = 'ANOTHER_ASYNC_ACTION'
const STARTED_ASYNC_ACTION = 'STARTED_ASYNC_ACTION'
const FINISHED_ASYNC_ACTION = 'FINISHED_ASYNC_ACTION'

// action creators
export const myAction = createAction(MY_ACTION)
export const myAsyncAction = createAction(MY_ASYNC_ACTION)
export const anotherAsyncAction = createAction(ANOTHER_ASYNC_ACTION)
const startedAsyncAsyncAction = createAction(STARTED_ASYNC_ACTION)
const finishedAsyncAsyncAction = createAction(FINISHED_ASYNC_ACTION)

// sagas
function * mySaga ({ payload }) {
  yield delay(1000) // <-- returns an effect that proceeds after a delay
  yield put(myAction(payload)) // <-- returns an effect that calls an action
}

function * anotherSaga ({ payload }) {
  const started = yield put(startedAsyncAction(payload)) // <-- mark the start of async
  yield delay(1000)
  yield put(myAction(payload))
  yield put(finishedAsyncAction(started.payload)) // <-- mark the end of async
}

// combine sagas
export const rootSaga = watchActions({
  // combine all of your module's sagas
  [MY_ASYNC_ACTION]: mySaga, // <-- calls the mySaga generator on MY_ASYNC_ACTION
  [ANOTHER_ASYNC_ACTION]: anotherSaga
})

// reducers
const myReducer = handleActions({
  [MY_ACTION]: (state, { payload }) => ({
    ...state,
    result: payload
  }),
  [STARTED_ASYNC_ACTION]: (state, action) => ({
    ...state,
    pending: true
  }),
  [FINISHED_ASYNC_ACTION]: (state, action) => ({
    ...state,
    pending: false
  })
}, {})

// combine reducers
export default combineReducers({
  // combine all of your module's reducers
  myReducer // <-- receives state.myReducer as state
})
```

## Add sagas to the todos module
If that generic module above is a little confusing that's ok. It's just boilerplate to capture some of the things you typically do in a module. With regards to our todo app we'll just use the [todos module we created in the previous tutorial]((./react-redux-starter-kit-todos.md#todos-module)). We'll be making some changes. We'll start with the finished module and then explain it piece by piece below.

You should note that right now the async portion of this is totally superficial -- we're simply adding a `delay()` instead of actually syncing to a server. It's important not to get too hung up on the server part of the transaction yet. We'll get deeper into using this with an API in the next tutorial.

Here is a full version of the todos module that utilizes [reselect](https://github.com/reactjs/reselect), [redux-actions](https://github.com/acdlite/redux-actions) and [redux-saga](https://github.com/yelouafi/redux-saga) to recreate the [todos app](http://redux.js.org/docs/basics/index.html). There's a lot going on here and you can start to see why some developers prefer to break their modules into smaller files.

If you're skimming for changes we're going to be adding a slight wait after you add a new todo. This will put the new todo in a pending state for 1 second before adding it to the store.

We'll go through this in detail below.

##### `src/routes/Todos/modules/todos.js`

```js
import { combineReducers } from 'redux'
import { createAction, handleActions } from 'redux-actions'
import { createSelector } from 'reselect'
import { v4 as uuid } from 'node-uuid'
import { delay } from 'redux-saga'
import { put } from 'redux-saga/effects'
import { watchActions } from '../../../store/sagas'

// selectors
export const getAppState = (state) => state.todosApp
export const getVisibilityFilter = (state) => getAppState(state).visibilityFilter
export const getTodos = (state) => getAppState(state).todos
export const getPendingTodos = (state) => getAppState(state).pendingTodos

export const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)

// constants
export const ADD_TODO_ASYNC = 'ADD_TODO_ASYNC'
const ADD_PENDING_TODO = 'ADD_PENDING_TODO'
const ADD_TODO = 'ADD_TODO'
const REMOVE_PENDING_TODO = 'REMOVE_PENDING_TODO'
export const SET_VISIBILITY_FILTER = 'SET_VISIBILITY_FILTER'
export const TOGGLE_TODO = 'TOGGLE_TODO'

// action creators
export const addTodoAsync = createAction(ADD_TODO_ASYNC)
const addPendingTodo = createAction(ADD_PENDING_TODO, text => ({ id: uuid(), text }))
const addTodo = createAction(ADD_TODO, text => ({ id: uuid(), text }))
const removePendingTodo = createAction(REMOVE_PENDING_TODO)
export const setVisibilityFilter = createAction(SET_VISIBILITY_FILTER)
export const toggleTodo = createAction(TOGGLE_TODO)

// sagas
export function * addTodoAsyncSaga ({ payload }) {
  const pending = yield put(addPendingTodo(payload))
  yield delay(1000)
  yield put(addTodo(payload))
  yield put(removePendingTodo(pending.payload))
}

// combine sagas
export const rootSaga = watchActions({
  [ADD_TODO_ASYNC]: addTodoAsyncSaga
})

// Reducers
const todo = handleActions({
  [ADD_TODO]: (state, { payload }) => ({
    id: payload.id,
    text: payload.text,
    completed: false
  }),
  [TOGGLE_TODO]: (state, { payload }) => (
    state.id !== payload ? state : {
      ...state,
      completed: !state.completed
    }
  )
})

export const todos = handleActions({
  [ADD_TODO]: (state, action) => ([
    ...state,
    todo(undefined, action)
  ]),
  [TOGGLE_TODO]: (state, action) => state.map(t => todo(t, action))
}, [])

export const visibilityFilter = handleActions({
  [SET_VISIBILITY_FILTER]: (state, { payload }) => payload
}, 'SHOW_ALL')

const pendingTodo = handleActions({
  [ADD_PENDING_TODO]: (state, { payload }) => ({
    id: payload.id,
    text: payload.text,
    completed: false,
    pending: true
  })
})

export const pendingTodos = handleActions({
  [ADD_PENDING_TODO]: (state, action) => ([
    ...state,
    pendingTodo(undefined, action)
  ]),
  [REMOVE_PENDING_TODO]: (state, { payload }) => state.filter(t => t.id !== payload.id)
}, [])

// Combined Reducer
export default combineReducers({
  todos,
  visibilityFilter,
  pendingTodos
})
```

## Using Selectors
*Note:* This section does a deep dive on using selectors. To follow along with sagas, skip to ["Using Sagas"](./redux-sagas-todos.md#using-sagas) below.

You can see above that we're using selectors for the first time in this tutorial series. At their core, selectors are functions that return a value from a specific part of the state. This helps formalize how your app interacts with the state. For instance, if you're storing the `visibilityFilter` under `state.todosApp.visibilityFilter` you might find it unsettling to paste that into every part of your app that needs to read the current visibility filter. It's easier to provide a simple accessor function.

You can see below that a selector is just a function that returns part of the state. You can easily chain your selectors. It's good practice to provide a generic `getAppState(state)` selector so that you could easily "move" your app in the redux store without having to refactor your entire app.

#### A simple selector in a module
A simple selector is just a function. It shouldn't perform any action. It should simply return a value from the state.

```js
// a simple selector in a module

// we provide an appState selector
// if state.todosApp needed to change we'd only have to update it here
export const getAppState = (state) => state.todosApp

// we use the appState selector to make our app's selectors easier to refactor
// this selector just returns the visibilityFilter from the todo app's state
export const getVisibilityFilter = (state) => getAppState().visibilityFilter

// you are intended to use a selector inside of mapStateToProps in a container component
// mapStateToProps take two arguments: state, containerProps
// standard selector functions take two arguments: state, props
// (it's not common to use the props argument unless you're doing something clever)
export const standardSelector = (state, props) => state.some.key.path
```

#### Using a selector in a container
From a container you use it like this:

```js
// ... snippet from src/routes/Todos/containers/FilterLink.js

// import a selector into a container
import { getVisibilityFilter } from '../modules/todos'

// you use selectors inside here
const mapStateToProps = (state, ownProps) => {
  return {

    // we pass the state to our selector, it returns the value we're looking for
    active: ownProps.filter === getVisibilityFilter(state)
  }
}
```

#### Creating memoized selectors with reselect
In practice most of the selectors you create will just be plain functions. When you need to combine selectors in complex ways, like if you're filtering a list, it's best practice to [memoize your selector](http://stackoverflow.com/questions/32543277/implementing-and-understanding-memoize-function-in-underscore-lodash). "[Memoizing](https://addyosmani.com/blog/faster-javascript-memoization/)" a function means that you cache the results of the function so that you only perform an expensive operation when you need to. This is a standard pattern and selectors are the ideal usecase to apply it. Helpfully, the [reselect](https://github.com/reactjs/reselect) makes it easy to compose selectors and memoize them in a standard way. Similar to using `createAction` from redux-actions, reselect provides a `createSelector` function. You should read the reselect Github page for more information.

Read about [memoization in the good parts](https://www.safaribooksonline.com/library/view/javascript-the-good/9780596517748/ch04s15.html).

```js
// ... selectors snippet from src/routes/Todos/modules/todos.js

import { createSelector } from 'reselect'

// Selectors
export const getAppState = (state) => state.todos
export const getVisibilityFilter = (state) => getAppState(state).visibilityFilter
export const getTodos = (state) => getAppState(state).todos

// this is a memoized selector function
// it calculates an expensive result by combining multiple selectors
export const getVisibleTodos = createSelector(

  // pass in an array of selector functions this memoized selector depends on
  // our memoized selector will only refresh if the results of any selector function changes
  // each selector is passed (state, containerProps)
  [ getVisibilityFilter, getTodos ],

  // the results of each selector are passed as arguments
  // these selector results are used to automanage the cache for the memoize function
  (visibilityFilter, todos) => {

    // perform a potentially  expensive action on a list
    // the return value is automatically cached by the memoize function
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed) // <-- Array.prototype.filter is considered "expensive"
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)
```

#### Using a reselect selector in a container
While a reselect selector looks more complicated in our module it's just as easy to use as a standard selector function.

```js
// ... snippet from src/routes/Todos/containers/VisibleTodoList.js

// import a reselect selector into a container
import { getVisibleTodos } from '../modules/todos'

const mapStateToProps = (state) => {
  return {

    // pass the state, get back a memoized result
    // the result stays fresh as the store updates
    todos: getVisibleTodos(state)
  }
}
```

## Using Sagas
We did a fairly thorough job of exploring sagas above. It might seem complicated but in practice sagas are really easy. We're going to review a saga for adding a new todo with a slight delay. Why? Because introducing a delay is enough to show how much a saga can do for us. When you're working with async actions it's good practice to track the progress in the app. We're going to call that 'pending' because any time we add a new todo we will wait 1 second before actually adding it. We're going to be updating our app in a few places to make it obvious how elegant the solution is. You may want to read about [how `yield *` works](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*).

```js
// ... sagas snippet from src/routes/Todos/modules/todos.js

import { delay } from 'redux-saga'
import { put } from 'redux-saga/effects'
import { watchActions } from '../../../store/sagas'

// sagas

// a saga is just a generator function
// all it does is dispatch actions
// within a saga you need to yield effects
export function * addTodoAsyncSaga ({ payload }) {

  // put returns the action object created by the addPendingTodo action creator
  // put is the same as dispatch, but designed to work with a saga
  // put generates an effect
  const pending = yield put( addPendingTodo(payload) )

  // we use a delay to pretend we're autosaving the todo on the server
  // redux-saga comes with a delay function that pauses the function
  // delay also generates an effect, just like put
  yield delay(1000)

  // after 1 second we dispatch the normal addTodo action
  yield put( addTodo(payload) )

  // to wrap things up we clear our pending todo
  yield put( removePendingTodo(pending.payload) )
}

// combine sagas
export const rootSaga = watchActions({

  // subscribe to run a saga when an action occurs
  [ADD_TODO_ASYNC]: addTodoAsyncSaga
})
```

### Creating actions for our saga
We're going to be creating a few new actions to support the new async method of adding new todos. In the original app we simply created a new todo with the `addTodo()` action. Now we'll be able to create a pending todo with the `addTodoAsync()` action. The async version of the action will eventually yield the same results as the sync action, but it first creates a pending todo for 1 second. This is similar to the concept behind [an optimisitic write](https://github.com/ForbesLindesay/redux-optimist).

```js
// ... actions snippet from src/routes/Todos/modules/todos.js

import { createAction, handleActions } from 'redux-actions'
import { v4 as uuid } from 'node-uuid'

// constants

// we're adding a new async constant
export const ADD_TODO_ASYNC = 'ADD_TODO_ASYNC'

// we don't need to export these constants if we don't use them outside this file
const ADD_TODO = 'ADD_TODO'
const ADD_PENDING_TODO = 'ADD_PENDING_TODO'   // <-- start
const REMOVE_PENDING_TODO = 'REMOVE_PENDING_TODO' // <-- end

export const SET_VISIBILITY_FILTER = 'SET_VISIBILITY_FILTER'
export const TOGGLE_TODO = 'TOGGLE_TODO'

// action creators

// usage: addTodoAsync(text)
// we need to export this action creator
export const addTodoAsync = createAction(ADD_TODO_ASYNC)

// the other action creators are for internal use only and we don't need to export them
// it's good practice to only export what you intend people to use from the outside
// it's pretty easy to add export later if you find that you need to use it elsewhere

// usage: addPendingTodo(text)
// generate a uuid for each pending todo
// we use the id to track the pending todo
const addPendingTodo = createAction(ADD_PENDING_TODO, text => ({ id: uuid(), text }))

// usage: addTodo(text)
// generate a uuid for each todo we add
// we use a different uuid from the pending todo
// presumeably we'd get the ID from the server in a real app
const addTodo = createAction(ADD_TODO, text => ({ id: uuid(), text }))

// usage: removePendingTodo({ id })
const removePendingTodo = createAction(REMOVE_PENDING_TODO)

export const setVisibilityFilter = createAction(SET_VISIBILITY_FILTER)
export const toggleTodo = createAction(TOGGLE_TODO)
```

### Using an async action in a container component
Now that we've got our sagas watching for actions and we've got our actions created, we need to dispatch our actions from a container.

We use actions in the `mapDispatchToProps` portion of a container component. All you really need to do is dispatch an action that we've configured our rootSaga to listen for. If our rootSaga sees our async action it will kick off our saga. This makes it really easy to implement async functionality from a containers perspective. It's good practice to capture complex async logic in your module so that you can control how people interact with your data in a centralized place.

We'll see a more complete version of this below.

```js
// ... fake snippet from src/routes/Todos/containers/AddTodo.js

// import an async action in a container component
import { addTodoAsync } from '../modules/todos'

// you dispatch actions in here
const mapDispatchToProps = (dispatch) => {
  return {
    addTodo: (text) => {

      // when this gets dispatched our addTodoAsyncSaga will capture it
      dispatch( addTodoAsync(text) )
    }
  }
}
```

### Creating reducers
Because we're tracking pending todos separate from our normal todos, we need to add a few new reducers. The `pendingTodos` reducer handles actions that come from our saga.

```js
// ... reducers snippet from src/routes/Todos/modules/todos.js

import { createAction, handleActions } from 'redux-actions'

// reducers

// ...

// we add a reducer for creating a pending todo
const pendingTodo = handleActions({
  [ADD_PENDING_TODO]: (state, { payload }) => ({
    id: payload.id,
    text: payload.text,
    completed: false,
    pending: true // <-- pending todos have a pending prop set to true
  })
})

// we need to handle adding and removing pending todos
export const pendingTodos = handleActions({

  // we create a new pending todo and add it to our pending array
  [ADD_PENDING_TODO]: (state, action) => ([
    ...state,
    pendingTodo(undefined, action)
  ]),

  // we remove the pending todo from the pending array by id
  [REMOVE_PENDING_TODO]: (state, { payload }) => state.filter(t => t.id !== payload.id)
}, [])

// combine reducers
export default combineReducers({
  todos,
  visibilityFilter,

  // we need to combine our reducer with the others
  // this becomes state.todosApp.pendingTodos
  pendingTodos
})
```

### Using our pending todos in a container
Here's the more complete version of using our new actions in a container. In the `onSubmit` function we dispatch our async action instead of the sync action.

##### `src/routes/Todos/containers/AddTodo.js`
(compare to [previous `src/routes/Todos/containers/AddTodo.js`](./react-redux-starter-kit-todos.md#srcroutestodoscontainersaddtodojs))

```js
// ... real snippet from src/routes/Todos/containers/AddTodo.js

import { addTodoAsync } from '../modules/todos'

let AddTodo = ({ dispatch }) => {
  // ...

  const onSubmit = e => {
    // ...

    // the only change is to dispatch the async action instead of the sync action
    dispatch( addTodoAsync(input.value) )

    // ...
  }

  // ...
}

// ...
```

When that action is captured by the saga it creates a new pending todo in the store. We can then read the pending todos and the real todos from the store and display them as "visible" todos.

##### `src/routes/Todos/containers/VisibleTodoList.js`
(compare to [previous `src/routes/Todos/containers/VisibleTodoList.js`](./react-redux-starter-kit-todos.md#srcroutestodoscontainersvisibletodolistjs))

```js
// ... snippet from src/routes/Todos/containers/VisibleTodoList.js

// we moved the selectors from this file to the module
import { getVisibleTodos, getPendingTodos } from '../modules/todos'

const mapStateToProps = (state) => {

  // get our visible and pending todos
  const visible = getVisibleTodos(state)
  const pending = getPendingTodos(state)

  return {

    // merge the visible and pending todos into a single list
    todos: visible.concat(pending)
  }
}

// ...
```

### Seeing our pending todos in a component
We need to alter our `TodoList` and our `Todo` components. Note that our TodoList component doens't really care if a todo is pending or not. We merged them together in the container.

##### `src/routes/components/TodoList.js`
(compare to [previous `src/routes/components/TodoList.js`](./react-redux-starter-kit-todos.md#srcroutestodoscomponentstodolistjs))

```js
// ... snippet from src/routes/components/TodoList.js

TodoList.propTypes = {
  todos: PropTypes.arrayOf(PropTypes.shape({
    id: PropTypes.string.isRequired,
    completed: PropTypes.bool.isRequired,
    pending: PropTypes.bool, // <-- we need to add a pending PropType
    text: PropTypes.string.isRequired
  }).isRequired).isRequired,
  onTodoClick: PropTypes.func.isRequired
}

export default TodoList
```

Finally we add a label to all of our pending todos. There is no complex logic in the component. It simply shows a "pending" label and changes the styling when a todos pending attribute is `true`.

##### `src/routes/components/Todos.js`
(compare to [previous `src/routes/components/Todos.js`](./react-redux-starter-kit-todos.md#srcroutestodoscomponentstodosjs))

```jsx
import React, { PropTypes } from 'react'

const Todo = ({ onClick, completed, pending, text }) => (
  <li
    onClick={onClick}
    style={{
      textDecoration: completed ? 'line-through' : 'none',
      fontStyle: pending ? 'italic' : 'normal', // <-- change style when pending
      color: pending ? 'gray' : 'inherit'
    }}
  >
    {text}
    {pending ? ' - pending' : '' /* <-- we add a label if it's pending */}
  </li>
)

Todo.propTypes = {
  onClick: PropTypes.func.isRequired,
  completed: PropTypes.bool.isRequired,
  pending: PropTypes.bool, // <-- we need to add a pending PropType
  text: PropTypes.string.isRequired
}

export default Todo
```
