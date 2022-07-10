---
title: 'Thinking in Effects'
---

<Intro>

The code inside your Effect will re-run after any change to the values that your Effect depends on. This is why you need to approach writing Effects with a different mindset than writing event handlers. You will always start by writing the code inside your Effect first. Then you'll specify its dependencies according to the code you already wrote. Finally, if some dependency changes too often and causes your Effect to re-run more often than necessary, you'll have to adjust the code so that it *does not need* that dependency, and then follow the same steps again. It's not always obvious how to do this, so this page will help you learn common patterns and idioms.

</Intro>

<YouWillLearn>

- How writing Effects is different from writing event handlers
- What causes Effects to re-run
- How to choose the Effect dependencies
- How the fix the dependency linter errors
- How to prevent Effects from re-running too often

</YouWillLearn>

## Effects are reactive {/*effects-are-reactive*/}

Event handlers and Effects [serve different purposes.](/learn/synchronizing-with-effects#what-are-effects-and-how-are-they-different-from-events) An event handler lets you respond to a specific interaction. An Effect lets your component stay synchronized with an external system. This is why they behave differently:

* An event handler runs exactly once per interaction. If you click a button once, its click handler will run once.
* **An Effect re-runs after any change to the values it depends on.** For example, say your Effect connects to the chat room specified by the `roomId` prop. Then you must include `roomId` [in your Effect's dependencies](/learn/synchronizing-with-effects#step-2-specify-the-effect-dependencies). When your component is added to the screen, your Effect will *run*, connecting to the room with the initial `roomId`. When it receives a different `roomId`, your Effect will *[clean up](/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed)* (disconnecting from the previous room) and *run* again (connecting to the next room). When your component is removed, the Effect will *clean up* one last time.

**In other words, Effects are _reactive._** They "react" to the values from the component body (like props and state). This might remind you of how React always keeps the UI synchronized to the current props and state of your component. By writing Effects, you teach React to synchronize *other* systems to the current props and state.

**Writing reactive code like Effects requires a different mindset.** The most common problem you'll run into is that your Effect re-runs too often because a dependency you use inside of it changes too often. Your first instinct might be to omit that dependency from the list, but that's wrong. What you need to do in these cases is to edit the *rest* of the code to *not need* that dependency. On this page, you'll learn the most common ways to do that.

### Event handlers run on specific interactions {/*event-handlers-run-on-specific-interactions*/}

This `ChatRoom` component contains an event handler that sends a chat message:

```js
function ChatRoom() {
  // ...
  function handleSendClick() {
    sendMessage(message);
  }
  // ...
}
```

Imagine you want to find out what causes this event handler to run. You can't tell that by looking at the code *inside* the event handler. You would need to look for where this event handler *is being passed* to the JSX:

```js
<button onClick={handleSendClick}>
  Send
</button>
```

The snippet above tells you that the `handleSendClick` function runs when the user presses the "Send" button. This event handler is written for *this particular interaction.* If the `handleSendClick` function isn't passed or called anywhere else, you can be certain that it *only* runs when the user clicks this particular "Send" button.

What happens when you edit the event handler? Let's say you're adding a `roomId` prop to the `ChatRoom` component. Then you read the current value of `roomId` inside your event handler:

```js {1,4}
function ChatRoom({ roomId }) {
  // ...
  function handleSendClick() {
    sendMessage(roomId, message);
  }
  // ...
}
```

Although you edited the code *inside* the `handleSendClick` event handler, this doesn't change when it runs. It still runs only when the user clicks the "Send" button. **Editing the event handler does not make it run any less or more often.** In other words, the code inside event handlers isn't reactive--it doesn't re-run automatically.

### Effects run whenever synchronization is needed {/*effects-run-whenever-synchronization-is-needed*/}

In the same `ChatRoom` component, there is an Effect that sets up the chat server connection:

```js
function ChatRoom() {
  // ...
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => {
      connection.disconnect();
    };
  }, []); // ✅ All dependencies declared
  // ...
}
```

It dependencies are an empty `[]` array, so this Effect only runs "on mount," i.e. when the component is added to the screen. (Note there'll still be an extra `connect()` and `disconnect()` call pair in development, [here's why.](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development))

Now let's see what happens when you add a `roomId` prop and start using it inside your Effect:

```js {1,4}
function ChatRoom({ roomId }) {
  // ...
  useEffect(() => {
    const connection = createConnection(roomId);
    // ...
}
```

If you only change these two lines, you will introduce a bug. The `ChatRoom` component will connect to the initial `roomId` correctly. However, when the user picks a different room, nothing will happen. Can you guess why?

If your linter is [correctly configured for React,](/learn/editor-setup#linting) it will point you directly to the bug:

```js {6}
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, []); // 🔴 React Hook useEffect has a missing dependency: 'roomId'
  // ...
}
```

This looks like a React error, but really it's React pointing out a logical mistake in this code. As it's written now, this Effect is not "reacting" to the `roomId` prop. **Since your Effect now uses the `roomId` prop in its code, it no longer makes sense for it to run only "on mount."** The linter verifies that your dependencies match your code.

To fix this bug, follow the linter's suggestion and add the missing `roomId` dependency:

```js {6}
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // ✅ All dependencies declared
  // ...
}
```

When your component mounts, your Effect will *run*, connecting to the room with the initial `roomId`. When it receives a different `roomId`, your Effect will *[clean up](/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed)* (disconnecting from the previous room) and *run* again (connecting to the next room). When your component unmounts, the Effect will *clean up* one last time.

Let's recap what has happened:

1. You edited the code, potentially introducing a bug.
2. The linter found the bug and suggested a change *to make the dependencies match the Effect code*.
3. You accepted the linter's suggestion, which (correctly) *made your Effect run more often than before*.

You'll go through a process like this every time you edit an Effect.

### Every reactive value becomes a dependency {/*every-reactive-value-becomes-a-dependency*/}

Imagine you're adding a way to choose the chat server, and read the server URL in your Effect:

```js {3,6}
function ChatRoom({ roomId, selectedServerUrl }) {
  const { defaultServerUrl } = useContext(SettingsContext);
  const serverUrl = selectedServerUrl ?? defaultServerUrl;

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // 🔴 React Hook useEffect has a missing dependency: 'serverUrl'
  // ...
}
```

As the linter points out, this code has a bug. If the user selects a different server, nothing will happen. Also, if the user hasn't selected any server yet, but they change the default server in your app's Settings, nothing will happen either. In other words, your Effect uses a value that can change over time, but it ignores all updates to that value.

**Since the `serverUrl` variable is declared inside your component, it is a _reactive value._ It is recalculated on every render--whenever the props or state change. This is why you have to specify it as a dependency of your Effect:**

```js {9}
function ChatRoom({ roomId, selectedServerUrl }) {
  const { defaultServerUrl } = useContext(SettingsContext);
  const serverUrl = selectedServerUrl ?? defaultServerUrl;

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId, serverUrl]); // ✅ All dependencies declared
  // ...
}
```

Now, your Effect will re-run more often. It will re-run not only when the user selects a different room, but also when they select a different server. And this is correct! This is what makes Effects reactive--**when you edit an Effect's code, you also change how often it runs.** This is how it stays synchronized to the latest props and state.

## How to fix an Effect that re-runs too often? {/*how-to-fix-an-effect-that-re-runs-too-often*/}

When your Effect uses a reactive value, you must include it in the dependencies. This may cause problems:

* Sometimes, you want to only read *the latest value* of some dependency instead of "reacting" to its changes.
* Sometimes, you want to re-execute *different parts* of your Effect under different conditions.
* Sometimes, a dependency may change too often *unintentionally* because it's an object or a function.

After you adjust the Effect's dependencies to reflect the code, you should always look at the dependency list. Does it make sense for the Effect to re-run when these dependencies change? Sometimes, the answer is "no."

**When this happens, your first instinct might be to omit some dependencies from the list, but that leads to subtle bugs that are very hard to diagnose. Instead, edit the *rest* of the code to *not need* that dependency.** It's not always obvious how to do this, so the rest of this page will introduce you to the most common scenarios.

### Removing an Effect {/*removing-an-effect*/}

The first thing you should think about is whether this code should be an Effect at all. For example, suppose you have a form thats submits a POST request and shows a notification [toast](https://uxdesign.cc/toasts-or-snack-bars-design-organic-system-notifications-1236f2883023). You trigger the Effect by setting state:

```js {4-9}
function Form() {
  const [submitted, setSubmitted] = useState(false);

  useEffect(() => {
    if (submitted) {
      post('/api/register');
      showToast('Successfully registered!');
    }
  }, [submitted]); // ✅ All dependencies declared

  function handleSubmit() {
    setSubmitted(true);
  }  

  // ...
}
```

Later, you want to style the notification toast according to the current theme, so you read the current theme. Since `theme` is declared in the component body, it is a reactive value, and you must declare it as a depedency:

```js {3,8,10}
function Form() {
  const [submitted, setSubmitted] = useState(false);
  const theme = useContext(ThemeContext);

  useEffect(() => {
    if (submitted) {
      post('/api/register');
      showToast('Successfully registered!', { theme });
    }
  }, [submitted, theme]); // ✅ All dependencies declared

  function handleSubmit() {
    setSubmitted(true);
  }  

  // ...
}
```

But by doing this, you've introduced a bug. Imagine you submit the form first and then switch between Dark and Light themes. The `theme` will change, the Effect will re-run, and so it will display the same notification again!

**The problem here is that this shouldn't be an Effect in the first place.** You want to send this POST request and show the notification in response to *submitting the form,* which is a particular interaction. When you want to run some code in response to particular interaction, put that logic directly into the corresponding event handler:

```js {5-6}
function Form() {
  const theme = useContext(ThemeContext);

  function handleSubmit() {
    post('/api/register');
    showToast('Successfully registered!', { theme });
  }  

  // ...
}
```

Now the code is inside the event handler, it's not reactive--so it will only run when the user submits the form.

Don't try to remove all Effects. It makes sense to write an Effect when you want to run some logic *because the component is displayed* rather than in response to a particular interaction. Read [Synchronizing with Effects](/learn/synchronizing-with-effects) to learn when to use Effects. Then, check out [You Might Not Need an Effect](/learn/you-might-not-need-an-effect) to learn when and how to avoid them.

### Splitting an Effect in two {/*splitting-an-effect-in-two*/}

Imagine you're creating a shipping form where the user needs to choose their city and area. You fetch the list of `cities` from the server according to the selected `country` so that you can show them as dropdown options:

```js
function ShippingForm({ country }) {
  const [cities, setCities] = useState([]);
  const [city, setCity] = useState(null);

  useEffect(() => {
    let ignore = false;
    fetchCities(country).then(json => {
      if (!ignore) {
        setCities(json);
      }
    });
    return () => {
      ignore = true;
    };
  }, [country]); // ✅ All dependencies declared

  // ...
```

This is a good example of [fetching data in an Effect.](/learn/you-might-not-need-an-effect#fetching-data) You are synchronizing the `cities` state with the network according to the `country` prop. You can't do this in an event handler because you need to fetch as soon as `ShippingForm` is displayed and whenever the `country` changes (no matter which interaction causes it).

Now let's say you're adding a second select box for city areas, which should fetch the `areas` for the currently selected `city`. You could try adding a `fetchAreas(city)` call to the same Effect when some `city` is selected:

```js {6,14-20}
function ShippingForm({ country }) {
  const [cities, setCities] = useState([]);
  const [city, setCity] = useState(null);
  const [areas, setAreas] = useState([]);

  // 🔴 Avoid: A single Effect synchronizes two independent processes
  useEffect(() => {
    let ignore = false;
    fetchCities(country).then(json => {
      if (!ignore) {
        setCities(json);
      }
    });
    if (city !== null) {
      fetchAreas(city).then(json => {
        if (!ignore) {
          setAreas(json);
        }
      });
    }
    return () => {
      ignore = true;
    };
  }, [country, city]); // ✅ All dependencies declared

  // ...
```

However, since the Effect now uses the `city` state variable, you've had to add `city` to the list of dependencies. That, in turn, has introduced a problem. Now, whenever the user selects a different city, the Effect will re-run and call `fetchCities(country)`. As a result, you will be unnecessarily refetching the list of cities many times.

**The problem with this code is that you're synchronizing two different unrelated things:**

1. You want to synchronize the `cities` state to the network based on the `country` prop.
1. You want to synchronize the `areas` state to the network based on the `city` state.

Split the logic into two Effects, each of which reacts to the prop that it needs to synchronize with:

```js {3-13,17-30}
function ShippingForm({ country }) {
  const [cities, setCities] = useState([]);
  useEffect(() => {
    let ignore = false;
    fetchCities(country).then(json => {
      if (!ignore) {
        setCities(json);
      }
    });
    return () => {
      ignore = true;
    };
  }, [country]); // ✅ All dependencies declared

  const [city, setCity] = useState(null);
  const [areas, setAreas] = useState([]);
  useEffect(() => {
    if (city === null) {
      return;
    }
    let ignore = false;
    fetchAreas(city).then(json => {
      if (!ignore) {
        setAreas(json);
      }
    });
    return () => {
      ignore = true;
    };
  }, [city]); // ✅ All dependencies declared

  // ...
```

After you split the Effects, changing the `city` no longer causes the `countries` to be refetched. Although [writing a chain of Effects that update state is unnecessary in synchronous code,](/learn/you-might-not-need-an-effect#chains-of-computations) it makes sense here. Both Effects talk to the network and are independent from each other. If you delete the first Effect and hardcode `city` to be `"Tokyo"`, the second Effect wouldn't break. Conversely, the first Effect still works if you delete the second one.

In the above example, both of the Effects you've split ended up looking very similar. However, the main reason you split them was because they *synchronize different processes.* A good rule of thumb is to check whether you can split the logic in a way that each Effect still works and makes sense if you delete the other one. If they work independently from each other or synchronize with different systems, splitting them apart usually makes sense.

<DeepDive title="Extracting independent Effects into custom Hooks">

When your component contains multiple Effects that are independent from each other, it's often a good idea to wrap those Effects into custom Hooks with a higher-level name and purpose. For example, you can move the logic to fetch the list of options for the dropdown to a custom `useFetchedList` Hook:

```js {4,6,10}
import { fetchCities, fetchAreas } from './api.js';

function ShippingForm({ country }) {
  const cities = useFetchedList(fetchCities, country);
  const [city, setCity] = useState('');
  const areas = useFetchedList(fetchAreas, city, city !== null);
  // ...
}

function useFetchedList(fetchList, parentId, shouldFetch = true) {
  const [options, setOptions] = useState([]);
  useEffect(() => {
    if (shouldFetch) {
      let ignore = false;
      fetchList(parentId).then(json => {
        if (!ignore) {
          setOptions(json);
        }
      });
      return () => {
        ignore = true;
      };
    }
  }, [fetchList, parentId]); // ✅ All dependencies declared
  return options;
}
```

This code is equivalent to the earlier example, but the person working on the `ShippingForm` component no longer needs to think about how these Effects work, and can focus on the purpose (fetch a list).

Note how `fetchList` must be a dependency now. Previously, this wasn't necessary because both your Effects directly used the `fetchCities` and `fetchAreas` imports from the top-level scope. Values from the top-level scope don't participate in the React rendering data flow, so you didn't need to include them as dependencies. However, now that you may pass an arbitrary function to `useFetchedList` as the `fetchList` argument, the linter makes sure that your Effect handles `fetchList` changing over time.

There is a concrete reason why this matters. Imagine someone extends your code a hundred years later:

```js {5,6}
import { fetchEarthCountries, fetchMoonCountries } from './api.js';

function ShippingForm() {
  const [isMoon, setIsMoon] = useState(false);
  const fetchCountries = isMoon ? fetchMoonCountries : fetchEarthCountries;
  const countries = useFetchedList(fetchCountries);
  // ...
}

```

If the user toggles the checkbox and calls `setIsMoon(true)`, the `useFetchedList` Hook will receive the `fetchMoonCountries` function rather than the `fetchEarthCountries` function as the `fetchList` argument. If you didn't include `fetchList` in the dependencies (which the linter makes you do), then this component would get "stuck" showing the Earth countries even after you've selected the Moon.

However, there is a pitfall. If you pass an *inline function* to `useFetchedList`, it will enter an infinite loop:

```js
function ShippingForm() {
  const cities = useFetchedList((c) => fetchCities(c), country);
```

Unlike previously, where the `fetchList` function was always an import (which doesn't change over time), when you pass an inline function, it will be a different function on every render. As a result, each time the `ShippingForm` component re-renders, the `fetchList` dependency will re-trigger the Effect. This will keep repeating because the Effect will set the state, leading to another re-render, and so on.

There are different ways you can solve this, depending on the API you prefer:

* You could make `useFetchedList` take a URL like `'/api/cities'` instead of an async function.
* You could warn in development if `useFetchedList` receives different functions over time.
* You could wrap the `useFetchedList` definition into a function called `createFetchableList` that takes the `fetchList` function and returns a `useFetchedList` Hook *for that specific kind of list.* Then you would write code like `const useFetchCountries = createFetchableList(fetchCountries)` at the top level outside of your component. This would ensure that the fetching function never changes.

You will learn more about how to safely call functions from Effects later on this page.

</DeepDive>

### Replacing objects and functions with primitives {/*replacing-objects-and-functions-with-primitives*/}

TODO

### Moving objects and functions outside the component {/*moving-objects-and-functions-outside-the-component*/}

TODO

### Moving objects and functions inside the Effect {/*moving-objects-and-functions-inside-the-effect*/}

TODO

### Updating state based on previous state from Effects {/*updating-state-based-on-previous-state-from-effects*/}

TODO

### Updating state based on other state from Effects {/*updating-state-based-on-other-state-from-effects*/}

TODO

### Calling event handlers from Effects {/*calling-event-handlers-from-effects*/}

TODO

### Reading the latest props and state {/*reading-the-latest-props-and-state*/}

TODO

## Recap {/*recap*/}

TODO

## Challenges {/*challenges*/}

TODO