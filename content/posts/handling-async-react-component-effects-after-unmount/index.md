---
date: 2021-01-24
title: Handling async React component effects after unmount
shortDescription: Four strategies for ensuring that React components don't update state after they've been unmounted
category: React
tags: [react, hooks, async, effects, unmount]
hero: ./empty-room-chair-serge-le-strat-tnw3krPFXxk-unsplash.jpg
heroAlt: An empty room with a lone chair
heroCredit: 'Photo by [Serge Le Strat](https://unsplash.com/@slestrat)'
---

Have you ever gotten this warning while developing React components?

```
Warning: Can't perform a React state update on an unmounted component.
This is a no-op, but it indicates a memory leak in your application.
To fix, cancel all subscriptions and asynchronous tasks in a
useEffect cleanup function.
```

This occurs when we try to update the state of a React component after it has been unmounted and removed from the component tree. And that is usually the result of making an async request (usually a data fetch), **but before the response is received and the data is stored in component state, the component has already been unmounted**.

I typically see these warnings during my Jest unit test runs (with [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)). Of course I mock out my `fetch` requests. But if I have a test run that just cares about the initial render, the test finishes and unmounts the test component before the mocked async action finishes.

It can be frustrating because it seems to never happen in our development environments and only tests, which aren't real speed use cases. But I believe the tests may be exposing potential production issues. **There are lots of variables that can cause a `fetch` request to be slower than expected**, so if the UI unmounts before the response returns, our users can find themselves in this situation.

Typically this won't break our apps because React will just ignore our state update function call, but this is still a warning that we want to prevent from happening. So how do we do that? How do we prevent it?

First lets look at a sample "app" that causes this error.

```js
import { useState, useEffect } from 'react'

const Results = () => {
  const [items, setItems] = useState([])

  useEffect(() => {
    // highlight-next-line
    fetchItems().then(setItems)
  }, [])

  return (
    <ol>
      {items.map((item) => (
        <li key={item}>{item}</li>
      ))}
    </ol>
  )
}

const App = () => {
  const [shown, setShown] = useState(false)

  return (
    <div>
      // highlight-next-line
      {shown && <Results />}
      <button onClick={() => setShown((curShown) => !curShown)}>
        {shown ? 'Hide' : 'Show'}
      </button>
    </div>
  )
}
```

[Try on CodeSandbox](https://codesandbox.io/s/elastic-knuth-eb1ly?file=/src/App.js)

In the example app, `<Results />` is rendered based upon the toggle state of the button. When the button is toggled off (using the [functional update](https://reactjs.org/docs/hooks-reference.html#functional-updates) form), `<Results />` is unmounted.

The `Results` component itself, asynchronously fetches items, and then calls `setItems` (the state updater) when the data comes back. **Now, if the button is toggled off after the call to `fetchItems`, but before the fetched data has returned, it will be unmounted when `setItems` called.** And that generates the warning. Try it out and click the button quickly. You'll see the warning in the CodeSandbox console.

Now let's get into the options for fixing.

## Option 1 - Variable to track mounted state

Vanilla [JavaScript Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) do not have the ability to be cancelled. So the next best alternative to avoid the React warning is to **not call the state updater if the component has been unmounted**. And in order to do _that_ we need to keep track of the mounted state.

This can be done inline in the same `useEffect` call in which we're making our `fetch` request.

```js
import { useState, useEffect } from 'react'

const Results = () => {
  const [items, setItems] = useState([])

  useEffect(() => {
    // highlight-next-line
    let mounted = true

    fetchItems().then((newItems) => {
      // highlight-next-line
      if (mounted) {
        setItems(newItems)
      }
    })

    // highlight-start
    return () => {
      mounted = false
    }
    // highlight-end
  }, [])

  // render UI
}
```

The `mounted` variable is initialized to `true` and then set to `false` in the [clean-up function](https://reactjs.org/docs/hooks-reference.html#cleaning-up-an-effect) returned by `useEffect`. That's how the mounted state is maintained. Then when the promise from `fetchItems()` resolves, we check to see if `mounted` is still `true`. If so, we'll call `setItems` with the new data. Otherwise, we'll do nothing. It's basically what React would do, but without the warning.

This local variable approach only works within a single `useEffect` call without any dependencies. It will not work across multiple `useEffect` calls within a component. It also isn't suitable for a `useEffect` call that has dependencies because the mounted state will be reset even though the component itself hasn't been unmounted.

## Option 2 - Ref to track mounted state

If we need to track the mounted state in multiple `useEffect` calls within a component, we can use a [ref](https://reactjs.org/docs/hooks-reference.html#useref) to maintain the mounted state across component re-renders.

```js
import { useState, useEffect, useRef } from 'react'

const Results = () => {
  const [items, setItems] = useState([])
  // highlight-next-line
  const mountedRef = useRef(false)

  // highlight-start
  // effect just for tracking mounted state
  useEffect(() => {
    mountedRef.current = true

    return () => {
      mountedRef.current = false
    }
  }, [])
  // highlight-end

  useEffect(() => {
    fetchItems().then((newItems) => {
      // highlight-next-line
      if (mountedRef.current) {
        setItems(newItems)
      }
    })
  }, [])

  useEffect(() => {
    // another fetch request for data
    // that will use `mountedRef.current`
  }, [])

  // render UI
}
```

The first `useEffect` call maintains the state with the `mountedRef`. It's set to `true` when the effect is first run and it's set to `false` when the component unmounts. Because the `useEffect` call has `[]` as its dependencies, it'll never run again when the `Results` component is re-rendered. But the `mountedRef` will continue to keep the mounted state across re-renders.

**Now, any other async effects can check the mounted state before calling state updaters because the ref is available everywhere within the component.**

## Option 3 - Custom Hook to track mounted state

If we need to track the mounted state in a number of different components, this is the perfect case to create a [custom Hook](https://reactjs.org/docs/hooks-custom.html). We basically want to move the logic from Option 2 into a custom Hook.

```js
import { useState, useEffect, useRef, useCallback } from 'react'

// returns a function that when called will
// return `true` if the component is mounted
const useMountedState = () => {
  const mountedRef = useRef(false)
  // highlight-next-line
  const isMounted = useCallback(() => mountedRef.current, [])

  useEffect(() => {
    mountedRef.current = true

    return () => {
      mountedRef.current = false
    }
  }, [])

  // highlight-next-line
  return isMounted
}

const Results = () => {
  const [items, setItems] = useState([])
  // highlight-next-line
  const isMounted = useMountedState()

  useEffect(() => {
    fetchItems().then((newItems) => {
      // highlight-next-line
      if (isMounted()) {
        setItems(newItems)
      }
    })
    // highlight-next-line
  }, [isMounted])

  // render UI
}
```

The `useMountedState` custom Hook uses the same ref to maintain the mounted state. However, it returns a function that when called returns the value of the ref. It leverages [`useCallback`](https://reactjs.org/docs/hooks-reference.html#usecallback) so that we don't recreate a new function every time `useMountedState` is called for every re-render of `Results`.

**Now, `useMountedState` can be called in any component, like `Results`, that needs to keep track of the mounted state.** And the `isMounted` function can be called multiple times within a given component as well.

Notice that the `isMounted` function becomes a new dependency for the `useEffect` call fetching the data. This is why it was important that `useMountedState` used `useCallback` to memoize the function. Otherwise, the fetch would be called with every re-render of `Results` because the function would be newly created with each re-render. For more details, read my post on [helper functions in the React `useEffect` Hook](/blog/helper-functions-react-useeffect-hook/).

By the way, the awesome [`react-use`](https://github.com/streamich/react-use) package (that contains every custom Hook imaginable) has the same [`useMountedState`](https://github.com/streamich/react-use/blob/master/docs/useMountedState.md) custom Hook.

## Option 4 - Custom Hook to fetch only when mounted

The previous three approaches provided us with a mounted state to do whatever we wished. But we were only using it in conjunction with making an async fetch request. So what if we created a custom Hook that specifically was for restricting async actions to the mounted state?

```js
const useSafeAsync = () => {
  const isMounted = useMountedState()
  // highlight-start
  const safeAsync = useCallback((promise) => {
    return new Promise((resolve) => {
      promise.then((value) => {
        if (isMounted()) {
          resolve(value)
        }
      })
    })
  })
  // highlight-end

  return safeAsync
}

const Results = () => {
  const [items, setItems] = useState([])
  // highlight-next-line
  const safeAsync = useSafeAsync()

  useEffect(() => {
    // highlight-start
    safeAsync(fetchItems()).then(setItems)
  }, [safeAsync])
  // highlight-end

  // render UI
}
```

Our `useSafeAsync` custom Hook is an abstraction of checking the mounted state after resolving a `Promise`. It makes use of the same `useMountedState` custom Hook to keep track of the mounted state and returns a function (`safeAsync`) that will take the `Promise` object returned from the async action. When that promise resolves, it verifies that the component is still mounted before passing along the resolution.

So now in our `Results` component, the code is very similar to the initial simple (but broken) code. **The only difference is we wrap the return value of `fetchItems()` in `safeAsync` so that we can know that anything we do afterward will happen when `Results` is still mounted.**

Notice this time, that `safeAsync` is listed as the dependencies of `useEffect`. That's why it's key that `useSafeAsync` uses `useCallback` to memoize the function it returns.

Once again, `react-use` has a similar custom Hook that it calls [`usePromise`](https://github.com/streamich/react-use/blob/master/docs/usePromise.md). It also handles promise rejection as well. [React Query](https://react-query.tanstack.com/), a library for fetching, caching and updating data, also has a [similar API](https://react-query.tanstack.com/guides/queries) while providing loads of additional functionality. If you're looking for a lower-level wrapper over the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API), but is still "safe", check out [`useFetch`](https://use-http.com/#/?id=basic-usage-auto-managed-state) from the [`use-http`](https://use-http.com/) package.

---

By the way, this warning can also happen if we update the state in an event listener handler, and we never remove the handler when the component is unmounted. This is actually a bigger problem because it means an event is being maintained for a component that no longer exists. **This is a memory leak.** The size of the memory leak of course depends on what event is being handled and how often its triggered.

But this is precisely what the clean-up return function of `useEffect` is for. **Always remember to remove event handlers you create.**

```js
const Example = () => {
  const [width, setWidth] = useState(0)

  useEffect(() => {
    const onResize = () => {
      setWidth(document.body.clientWidth)
    }

    // highlight-next-line
    window.addEventListener('resize', onResize)

    return () => {
      // highlight-next-line
      window.removeEventListener('resize', onResize)
    }
  }, [])

  return <span>Width: {width}</span>
}
```

Keep learning my friends. 🤓
