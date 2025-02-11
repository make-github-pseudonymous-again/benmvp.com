---
date: 2021-02-28
title: Sync to localStorage with React useReducer Hook
shortDescription: How to create a React custom Hook that wraps useReducer to persist state to localStorage
category: React
tags: [react, hooks, localStorage]
hero: ./floppy-disk-hosein-zanbori-FIFhOXkhAw4-unsplash.jpg
heroAlt: Red floppy disk
heroCredit: 'Photo by [hosein zanbori](https://unsplash.com/@hoseincameraman)'
---

In my last post on [React custom Hooks vs. Mixins](/blog/react-custom-hooks-mixins/), I compared a custom Hook I recently wrote with its equivalent in Mixins form. In this post now, I want to share a [`useReducer`](https://reactjs.org/docs/hooks-reference.html#usereducer) + [`localStorage`](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) custom Hook I needed to synchronize my reducer state to `localStorage`.

I Googled around for "useReducer localStorage" for the best approach to this and was surprised to find very little answers. I also wanted to be able to use the [`useLocalStorage`](https://github.com/streamich/react-use/blob/master/docs/useLocalStorage.md) Hook from [`react-use`](https://github.com/streamich/react-use) to handle the `localStorage` piece.

After coalescing different blog posts and StackOverflow answers, I came up with this `usePersistReducer` Hook:

```js
import { useCallback, useReducer } from 'react'
import { useLocalStorage } from 'react-use'

const LOCAL_STORAGE_KEY = 'KEY_GOES_HERE'

const INITIAL_STATE = {
  // initial state
}

const reducer = (state, action) => {
  // return updated state based on `action.type`
}

const usePersistReducer = () => {
  // grab saved value from `localStorage` and
  // a function to update it. if
  // no value is retrieved, use `INITIAL_STATE`
  const [savedState, saveState] = useLocalStorage(
    LOCAL_STORAGE_KEY,
    INITIAL_STATE,
  )

  // wrap `reducer` with a memoized function that
  // syncs the `newState` to `localStorage` before
  // returning `newState`. memoizing is important!
  const reducerLocalStorage = useCallback(
    (state, action) => {
      const newState = reducer(state, action)

      saveState(newState)

      return newState
    },
    [saveState],
  )

  // use wrapped reducer and the saved value from
  // `localStorage` as params to `useReducer`.
  // this will return `[state, dispatch]`
  return useReducer(reducerLocalStorage, savedState)
}

const Example = () => {
  // return value from `usePersistReducer` is identical
  // to `useReducer`
  const [state, dispatch] = usePersistReducer()

  // render UI based on `state`
  // call `dispatch` based on user actions
}
```

> I like to create custom Hooks, even if I'm using them in one place, to encapsulate logic together. It's nice to give a name to multiple Hooks used together. Here, I'm using `usePersistReducer` solely to wrap this particular `reducer` so it doesn't take any parameters. But if I wanted to make `usePersistReducer` a more reusable custom Hook, it would probably take a `reducer`, `storageKey`, and `initialState` as parameters.

And that's it! If all you were looking for was an answer, you don't need to read any further. But if you're interested in more details, by all means keep reading. 😄

---

Let's first look at the use of `useLocalStorage`:

```js
// grab saved value from `localStorage` and
// a function to update it. if
// no value is retrieved, use `INITIAL_STATE`
const [savedState, saveState] = useLocalStorage(
  LOCAL_STORAGE_KEY,
  INITIAL_STATE,
)
```

This one Hook does so much for us behind the scenes. It retrieves the string stored at the `LOCAL_STORAGE_KEY`, parses the JSON, and returns the result as `savedState`. If the key doesn't exist, it returns `INITIAL_STATE` in its place. `saveState` is a memoized function that will stringify an object to JSON and store it at the `LOCAL_STORAGE_KEY`.

Then there's `reducerLocalStorage`. This is where the magic happens:

```js
// wrap `reducer` with a memoized function that
// syncs the `newState` to `localStorage` before
// returning `newState`. memoizing is important!
const reducerLocalStorage = useCallback(
  (state, action) => {
    const newState = reducer(state, action)

    saveState(newState)

    return newState
  },
  [saveState],
)
```

The memoized `reducerLocalStorage` first calls the underlying reducer passing through the `state` and `action`. The gives us the new state. But before returning the new state, we call `saveState` to persist the state to `localStorage`.

This approach works for two reasons. First, [`useReducer`](https://reactjs.org/docs/hooks-reference.html#usereducer) only needs a function that accepts `state` & `action` as parameters, and returns an updated state based on the action. Typically that function is the reducer itself, but in our case, it's this "wrapped reducer" that looks and acts like a reducer.

The second reason is connected to the first: saving and reading from `localStorage` is synchronous. If we were synching to an external API, an asynchronous action, this code would be a bit more complex in order for it to act like a normal reducer.

```js
// use wrapped reducer and the saved value from
// `localStorage` as params to `useReducer`.
// this will return `[state, dispatch]`
return useReducer(reducerLocalStorage, savedState)
```

Lastly, we use the wrapped reducer function with the saved data from `localStorage` as the parameters passed to `useReducer`.

I mention it in the code comments earlier, but we must memoize `reducerLocalStorage` with `useCallback` for this to work. The `useReducer` Hook expects the reducer function passed to it to be constant across re-renders. We used the `saveState` function from `useLocalStorage` as the dependency in `useCallback`, but that function also is memoized (based on the storage key). So `reducerLocalStorage` should remain the same.

## Without the `useLocalStorage` Hook

The [`useLocalStorage`](https://github.com/streamich/react-use/blob/master/docs/useLocalStorage.md) Hook from the [`react-use`](https://github.com/streamich/react-use) library of helpful custom Hooks does a lot for us. But if you can't use it (or don't want to use it), we can create a `usePersistReducer` without it.

```js
const init = () => {
  // initialize state w/ existing value saved in `localStorage`
  try {
    const localStorageValue = localStorage.getItem(LOCAL_STORAGE_KEY)

    if (localStorageValue !== null) {
      // if there's an existing value in `localStorage`,
      // parse it and return it to be initial reducer state
      return JSON.parse(localStorageValue)
    } else {
      // if no existing value exists, we'll use `INITIAL_STATE`.
      // but sync this backup value to `localStorage` first
      localStorage.setItem(LOCAL_STORAGE_KEY, INITIAL_STATE)
    }
  } catch {
    // if user is in incognito mode, `localStorage` access
    // will throw an error. if the value is malformed JSON
    // `JSON.parse` will throw an error as well
  }

  return INITIAL_STATE
}

const usePersistReducer = () => {
  // Wrap `reducer` with a memoized function that
  // syncs the `newState` to `localStorage` before
  // returning `newState`. Memoizing is important!
  const reducerLocalStorage = useCallback(
    (state, action) => {
      const newState = reducer(state, action)

      try {
        // store new state in local storage
        localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(newState))
      } catch {
        // if user is in incognito mode, `localStorage` access
        // will throw an error. The state could fail to stringify
        // too. So do nothing
      }

      return newState
    },
    [LOCAL_STORAGE_KEY],
  )

  // use `init` function to initialize state from `localStorage`
  return useReducer(reducerLocalStorage, undefined, init)
}
```

As you can see it's not quite as "clean." The storage retrieval/updating is separated in two places. I guess we could've created our own `useLocalStorage`. But anyway, this is still better than having all of this code within the component itself.

## With TypeScript

I pretty much use TypeScript in all the JavaScript code I write nowadays. I find TypeScript to be especially helpful with data transformation functions because it helps speed up development knowing exactly what my types are.

If you're getting into TypeScript, I have an existing post on [using TypeScript with `useReducer`](https://www.benmvp.com/blog/type-checking-react-usereducer-typescript/) which explains adding types for the `reducer` function. So here's the little bit needed to add types for `usePersistReducer`:

```typescript
const usePersistReducer = () => {
  const [savedState, saveState] = useLocalStorage(
    LOCAL_STORAGE_KEY,
    INITIAL_STATE,
  )
  const reducerLocalStorage = useCallback(
    // give `reducerLocalStorage` the same TS API
    // as the underlying `reducer` function
    // highlight-next-line
    (state: State, action: Action): State => {
      const newState = reducer(state, action)

      saveState(newState)

      return newState
    },
    [saveState],
  )

  return useReducer(reducerLocalStorage, savedState)
}
```

---

That's it! If `react-use` didn't already have [over 100 PRs](https://github.com/streamich/react-use/pulls) waiting to be merged, I'd create one for `usePersistReducer`. But for now, we have something we can use.

Keep learning my friends. 🤓
