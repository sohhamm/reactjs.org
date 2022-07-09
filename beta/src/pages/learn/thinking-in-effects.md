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
  }, []);
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

### Splitting the Effect in two {/*splitting-the-effect-in-two*/}


