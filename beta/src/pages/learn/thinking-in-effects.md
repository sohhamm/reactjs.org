---
title: 'Thinking in Effects'
---

<Intro>

An Effect re-runs after any change to the values that it depends on. This is why writing Effects requires a different mindset than writing event handlers. You'll write the code inside your Effect first. Then you'll specify its dependencies according to the code you already wrote. Finally, if a dependency changes too often and causes your Effect to re-run more than needed, you'll have to adjust the code so that it *does not need* that dependency, and follow these steps again. It's not always obvious how to do this, so this page will help you learn common patterns and idioms.

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
  }, []);
  // ...
}
```

Its dependencies are an empty `[]` array, so this Effect only runs "on mount," i.e. when the component is added to the screen. (Note there'll still be an extra `connect()` and `disconnect()` call pair in development, [here's why.](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development))

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

<DeepDive title="What kind of values can be dependencies?">

Only values that participate in the rendering data flow--and therefore could change over time--are dependencies. This includes every value that's defined **directly inside the component**, such as props, state, context, and other variables that are directly inside the component and are used by the Effect:

```js
// 🔴 Variables outside the component can't be dependencies
const notADependency1 = // ...

function YourComponent({ dependency1 }) { // ✅ Props can be dependencies
  // ✅ Variables directly inside the component can be dependencies:
  const dependency2 = // ...
  const [dependency3, dependency4] = useSomething();

  useEffect(() => {
    // 🔴 Variables inside the Effect can't be dependencies.
    const notADependency2 = // ...
    // ...
  }, [dependency1, dependency2, dependency3, dependency4]);
  // ...
}
```

Global or mutable values can't be dependencies:

```js
function Chat() {
  const ref = useRef(null);
  useEffect(() => {
    // ...
  }, [
    window.location.query, // 🔴 Can't be a dependency: mutable and global
    ref.current.offsetTop  // 🔴 Can't be a dependency: mutable
  ]);
}
```

They're not valid dependencies because they don't participate in the React rendering flow. Changing a mutable value doesn't trigger a re-render, so React wouldn't know to re-run the Effect. However, if there is a way to subscribe to the changes of the mutable value you're interested in, you can [synchronize it with the React state](/learn/you-might-not-need-an-effect#subscribing-to-an-external-store), and then use that React state variable as a dependency of your Effect.

Values that are guaranteed to be the same on every render can be omitted. For example, the `ref` wrapper object returned from [`useRef`](/apis/useref) and the `set` function returned by [`useState`](/apis/usestate) are *stable,* i.e. guaranteed to never change between re-renders by React. You can omit them from the list. However, the dependency linter may ask you to include them if it can't verify that they're directly coming from React (for example, if you pass them from a parent component). In that case you should specify them.

Objects and functions can be dependencies, but you need to be careful:

```js
function Page() {
  const obj = {}; // 🔴 Inline object will be different on every render
  function fn() { // 🔴 Inline function will be different on every render
    // ...
  }
  useEffect(() => {
    fn(obj);
  }, [obj, fn]);
  // ...
}
```

Inline objects and functions are always "new": [`{} !== {}`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality#comparing_objects) and `function(){} !== function(){}`. This makes the `fn` and `obj` dependencies above "always different," so the Effect will re-run after every render. To fix this, you can usually replace object and function dependencies with simpler primitive dependencies, or remove the need for them altogether. You'll learn how to do this [later on this page.](#how-to-fix-an-effect-that-re-runs-too-often)

</DeepDive>

## How to fix an Effect that re-runs too often? {/*how-to-fix-an-effect-that-re-runs-too-often*/}

When your Effect uses a reactive value, you must include it in the dependencies. This may cause problems:

* Sometimes, you want to re-execute *different parts* of your Effect under different conditions.
* Sometimes, a dependency may change too often *unintentionally* because it's an object or a function.
* Sometimes, you want to only read *the latest value* of some dependency instead of "reacting" to its changes.

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
      showToast('Successfully registered!', theme);
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
    showToast('Successfully registered!', theme);
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
  const [cities, setCities] = useState(null);
  const [city, setCity] = useState(null);

  useEffect(() => {
    let ignore = false;
    fetch(`/api/cities?country=${country}`)
      .then(response => response.json())
      .then(json => {
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

Now let's say you're adding a second select box for city areas, which should fetch the `areas` for the currently selected `city`. You might start by adding a second `fetch` call for the list of areas inside the same Effect:

```js {15-24,28}
function ShippingForm({ country }) {
  const [cities, setCities] = useState(null);
  const [city, setCity] = useState(null);
  const [areas, setAreas] = useState(null);

  useEffect(() => {
    let ignore = false;
    fetch(`/api/cities?country=${country}`)
      .then(response => response.json())
      .then(json => {
        if (!ignore) {
          setCities(json);
        }
      });
    // 🔴 Avoid: A single Effect synchronizes two independent processes
    if (city) {
      fetch(`/api/areas?city=${city}`)
        .then(response => response.json())
        .then(json => {
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

```js {19-32}
function ShippingForm({ country }) {
  const [cities, setCities] = useState(null);
  useEffect(() => {
    let ignore = false;
    fetch(`/api/cities?country=${country}`)
      .then(response => response.json())
      .then(json => {
        if (!ignore) {
          setCities(json);
        }
      });
    return () => {
      ignore = true;
    };
  }, [country]); // ✅ All dependencies declared

  const [city, setCity] = useState(null);
  const [areas, setAreas] = useState(null);
  useEffect(() => {
    if (city) {
      fetch(`/api/areas?city=${city}`)
        .then(response => response.json())
        .then(json => {
          if (!ignore) {
            setAreas(json);
          }
        });
    }
    return () => {
      ignore = true;
    };
  }, [city]); // ✅ All dependencies declared

  // ...
```

Now the first Effect only re-runs if the `country` changes, while the second Effect re-runs when the `city` changes. You've separated them by purpose: two separate Effects synchronize two different things.

The final code is longer than the original, but splitting these Effects is still correct. **Each Effect should represent an independent synchronization process.** If there is one thing being synchronized, there should be one Effect. If there are two different things being synchronized independently from each other, then there should be two Effects. You should split Effects according to their purpose, not whether the code is shorter or "feels cleaner."

In the above example, deleting one Effect wouldn't break the other Effect's logic. This is a good indication that they synchronize different things, and so it made sense to split them up. On the other hand, if you split up a cohesive piece of logic into separate Effects, the code may look "cleaner" but will be [more difficult to maintain.](/learn/you-might-not-need-an-effect#chains-of-computations)

### Wrapping an Effect into a custom Hook {/*wrapping-an-effect-into-a-custom-hook*/}

In the above example, the two Effects are independent from each other but share a lot of repetitive code. This makes the component itself difficult to read. You have to pause to figure out what exactly each Effect does, and how the data flows into and out of each Effect. This is especially difficult when asynchronous logic is involved.

You can simplify the `ShippingForm` component above by extracting the Effect into your own `useData` Hook:

```js {1}
function useData(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    if (url) {
      let ignore = false;
      fetch(url)
        .then(response => response.json())
        .then(json => {
          if (!ignore) {
            setData(json);
          }
        });
      return () => {
        ignore = true;
      };
    }
  }, [url]); // ✅ All dependencies declared
  return data;
}
```

Now you can replace both Effects in the `ShippingForm` components with calls to your custom `useData` Hook:

```js {2,4}
function ShippingForm({ country }) {
  const cities = useData(`/api/cities?country=${country}`);
  const [city, setCity] = useState(null);
  const areas = useData(city ? `/api/areas?city=${city}` : null);
  // ...
```

Custom Hooks like `useData` make your components easier to read and maintain:

- **Custom Hooks lets you emphasize the intent.** When your component body contained several raw Effects, it was tricky to tell at a glance how the data flowed in and out. Now that the logic is in the `useData` Hook, you can "forget" how it works and treat it as a black box: you feed the `url` in, and you get the `data` out.
- **You can reuse custom Hooks between components.** As you create more app-specific custom Hooks or import them from the community packages, you won't need to write Effects in your components as often.

Custom Hooks also make it easier to replace your Effects later. For example, if you decide to switch to a more efficient data fetching solution, it's less work to migrate from a Hook like `useData` than from the raw `useEffect` scattered across many different components. [Read more about data fetching with Effects and the alternatives.](/learn/you-might-not-need-an-effect#fetching-data)

<DeepDive title="When is extracting a custom Hook a good idea?">

Start by choosing your custom Hook's name. If you struggle to pick a clear name, it might mean that your Effect is too coupled to the rest of your component's logic, and is not yet ready to be extracted.

Ideally, your custom Hook's name should be clear enough that even a person who doesn't write code often could have a good guess about what your custom Hook does, what it takes, and what it returns:

* ✅ `useData(url)`
* ✅ `useImpressionLog(eventName, extraData)`
* ✅ `useChatConnection(roomId)`

When you synchronize with an external system, your custom Hook name may be more technical and use jargon specific to that system. It's good as long as it would be clear to a person familiar with that system:

* ✅ `useMediaQuery(query)`
* ✅ `useSocket(url)`
* ✅ `useIntersectionObserver(ref, options)`

**Keep custom Hooks focused on concrete high-level use cases.** Avoid creating and using custom "lifecycle" Hooks that act as alternatives and convenience wrappers for the `useEffect` API itself:

* 🔴 `useMount(fn)`
* 🔴 `useEffectOnce(fn)`
* 🔴 `useUpdateEffect(fn)`

For example, this `useMount` Hook tries to ensure some code only runs "on mount":

```js {2-3,12-13}
function ChatRoom() {
  // 🔴 Avoid: using custom "lifecycle" Hooks
  useMount(() => {
    const connection = createConnection();
    connection.connect();

    post('/analytics/event', { eventName: 'visit_chat' });
  });
  // ...
}

// 🔴 Avoid: creating custom "lifecycle" Hooks
function useMount(fn) {
  useEffect(() => {
    fn();
  }, []); // 🔴 React Hook useEffect has a missing dependency: 'fn'
}
```

**Custom "lifecycle" Hooks like `useMount` don't fit well into the React paradigm.** For example, if you used this `useMount` Hook instead of a raw `useEffect` in the earlier [chat room example](#effects-run-whenever-synchronization-is-needed), the linter wouldn't find the mistake in your code when you forgot to "react" to `roomId` changes. (And if you *don't* want some prop or state to cause the Effect to re-run, [there is a different recommended way to do that.](#reading-the-latest-props-and-state-from-effects))

Similarly, if you alias the `useEffect(fn, [])` pattern with a "nicer" name like `useEffectOnce`, React's [remounting components in development](/learn/synchronizing-with-effects#how-to-handle-the-effect-firing-twice-in-development) would make its name misleading. In general, if you find yourself trying to "work around" React's behavior in a custom Hook, it's time to pause and rethink the approach.

If you're writing an Effect, start by using the React API directly:

```js
// ✅ Good: raw Effects separated by purpose
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection();
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);

  useEffect(() => {
    post('/analytics/event', { eventName: 'visit_chat', roomId });
  }, [roomId]);

  // ...
}
```

Then, you can (but don't have to) extract custom Hooks for different high-level use cases:

```js
// ✅ Great: custom Hooks named after their purpose
function ChatRoom({ roomId }) {
  useChatConnection(roomId);
  useImpressionLog('visit_chat', { roomId });
  // ...
}
```

**A good custom Hook makes the calling code more declarative by constraining what it does.** For example, `useChatConnection(roomId)` can only connect to the chat room, while `useImpressionLog(eventName, extraData)` can only send an impression log to the analytics. If your custom Hook API doesn't constrain the use cases and is very abstract, in the long run it's likely to introduce more problems than it solves.

</DeepDive>

### Replacing objects and function dependencies with primitives {/*replacing-objects-and-function-dependencies-with-primitives*/}

TODO

### Moving objects and functions outside the component {/*moving-objects-and-functions-outside-the-component*/}

TODO

### Moving objects and functions inside the Effect {/*moving-objects-and-functions-inside-the-effect*/}

TODO

### Updating state from Effects {/*updating-state-based-on-previous-state-from-effects*/}

TODO

### Calling event handlers from Effects {/*calling-event-handlers-from-effects*/}

TODO

### Reading the latest props and state from Effects {/*reading-the-latest-props-and-state-from-effects*/}

TODO

## Recap {/*recap*/}

TODO

## Challenges {/*challenges*/}

TODO