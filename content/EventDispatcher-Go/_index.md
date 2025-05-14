+++
title = "Event Dispatcher for Go"
+++

## Events

When you dispatch an event, it is identified by a unique name (e.g. `"user.created"`). Each listener registered for that name will receive an `Event` instance, which may implement additional interfaces for advanced behavior.

### Basic Event

The simplest event type is the empty `Event` interface. For stoppable events, embed `BaseEvent`:

```go
type MyEvent struct {
    BaseEvent            // provides StopPropagation, IsPropagationStopped
    UserID    string     // custom payload
}

func (e *MyEvent) EventName() string {
    return "my_event"
}
```

- **StopPropagation**: Call `e.StopPropagation()` in a listener to prevent further listeners from running.
- **EventName**: Optional method to infer event name if you pass `""` to `Dispatch`.

---

## Dispatcher

The `EventDispatcher` is the central hub. It maintains per-event listener lists (sharded locks + copy-on-write).

```go
d := eventdispatcher.NewEventDispatcher()
```

- **Thread-safe**: uses `sync.Map` and per-event `RWMutex` for minimal contention.
- **Lock-free dispatch**: snapshot handler lists, then release lock before invoking.
- **Panic-safe**: recovers listener panics to ensure availability.

---

## Connecting Listeners

Register a listener function with optional priority:

```go
unsubscribe := d.AddListener("user.created",
    func(e eventdispatcher.Event) {
        evt := e.(*UserCreated)
        fmt.Println("User created:", evt.UserID)
    },
    10, // higher priority executes earlier
)
```

- **Parameters**:
  1. `eventName` (string)
  2. `Listener` (func(Event))
  3. `priority` (int, default `0`)

- **Unsubscribe**: call the returned `unsubscribe()` closure to remove this listener by unique ID.

- **RemoveListener**: remove by function pointer:

  ```go
  d.RemoveListener("user.created", yourFunc)
  ```

---

## Creating and Dispatching Events

### Creating Custom Event

Define an event struct embedding `BaseEvent` or implementing `EventName()`:

```go
type OrderPlacedEvent struct {
    BaseEvent
    OrderID string
}

func (e *OrderPlacedEvent) EventName() string {
    return "order.placed"
}
```

### Dispatching

Send the event:

```go
evt := &OrderPlacedEvent{OrderID: "1234"}
d.Dispatch(evt, "")  // uses EventName()
```

Or explicitly:

```go
d.Dispatch(evt, "order.placed")
```

---

## Generic Events & Pooling

For dynamic data or simple payloads, use `GenericEvent` with pooling:

```go
// Acquire from pool
ge := eventdispatcher.AcquireGenericEvent("dynamic.event", map[string]interface{}{
    "foo": 42,
})
// optional: ge.Subject = yourSubject
defer eventdispatcher.ReleaseGenericEvent(ge)

d.AddListener("dynamic.event", func(e eventdispatcher.Event) {
    gev := e.(*eventdispatcher.GenericEvent)
    fmt.Println("Arg foo:", gev.OffsetGet("foo"))
}, 0)

d.Dispatch(ge, "")
```

- **Arguments API**:
  - `GetArguments()`, `SetArguments(map)`
  - `HasArgument`, `GetArgument`, `SetArgument`
  - `OffsetExists`, `OffsetGet`, `OffsetSet`, `OffsetUnset`, `GetIterator`

- **Pooling**: reuse `GenericEvent` instances to reduce GC.

---

## Subscriber Pattern

Group multiple event handlers in one struct:

```go
type MySubscriber struct {
    called map[string]int
}

func (s *MySubscriber) SubscribedEvents() []eventdispatcher.Subscription {
    return []eventdispatcher.Subscription{
        {"user.created", s.onUserCreated, 0},
        {"order.placed", s.onOrderPlaced, 5},
    }
}

sub := &MySubscriber{called: make(map[string]int)}
unsubs := d.AddSubscriber(sub)

// Remove a single listener
unsubs[0]()

// Remove all subscriber listeners
d.RemoveSubscriber(sub)
```

- `AddSubscriber` returns individual unsubscribe closures if needed.
- `RemoveSubscriber` unsubscribes all registered handlers for that subscriber.

---

## Immutable Dispatcher

Create a **read-only** proxy that panics on mutators:

```go
imm := eventdispatcher.NewImmutableEventDispatcher(d)
imm.Dispatch(evt, "name")    // works
imm.AddListener("x", nil,0)  // panic
```

Use when you need to expose dispatcher without mutation rights.

---

## API Summary

- **EventDispatcher**
  - `NewEventDispatcher()`
  - `AddListener(name string, fn Listener, prio int) func()`
  - `RemoveListener(name string, fn Listener)`
  - `Dispatch(event Event, name string) Event`
  - `HasListeners(name string) bool`
  - `Listeners(name string) []Listener`
  - `AddSubscriber(sub Subscriber) []func()`
  - `RemoveSubscriber(sub interface{})`

- **GenericEvent**
  - `NewGenericEvent(subject interface{}, args map[string]interface{}) *GenericEvent`
  - `GetSubject()/SetSubject()`
  - `GetArguments()/SetArguments(map[string]interface{})`
  - `HasArgument/GetArgument()`
  - `SetArgument()`
  - `OffsetExists/OffsetGet/OffsetSet/OffsetUnset`
  - `GetIterator()`

- **Pooling**
  - `AcquireGenericEvent(name string, payload interface{}) *GenericEvent`
  - `ReleaseGenericEvent(*GenericEvent)`

- **ImmutableEventDispatcher**
  - `NewImmutableEventDispatcher(*EventDispatcher)`

- **Subscriber** & **Subscription**

---

## Examples

See full examples and tests in the `examples/` directory of the repository.

---

```go
// Complete sample
package main

import (
    "fmt"
    "github.com/lemric/eventdispatcher-go"
)

type MsgEvent struct {
    eventdispatcher.BaseEvent
    Msg string
}

func (e *MsgEvent) EventName() string { return "msg.event" }

func main() {
    d := eventdispatcher.NewEventDispatcher()
    d.AddListener("msg.event", func(evt eventdispatcher.Event) {
        e := evt.(*MsgEvent)
        fmt.Println("Received:", e.Msg)
    }, 0)
    d.Dispatch(&MsgEvent{Msg: "Hello"}, "")
}
```
