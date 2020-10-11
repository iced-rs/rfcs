# Introduction

As discussed in #89, users expect a _blinking_ cursor to signify the current insert position, of a focused text input.

However, implementing this is more complicated than simply updating the `TextInput` widget, because currently iced widgets can only update in response to user events, or application messages (via the Subscription system).

This document aims to walk through this problem space and propose a minimal solution to enable a blinking `TextInput`.

# Guide-level explanation

### Background on current architecture 

#### `TextInput` widget

Currently, the cursor is always shown (ie: it does not blink) for a focused text input. No cursor is shown for a `TextInput` which does not have focus.

All changes to the text-input's appearance are driven by user-interaction events.

#### The runtime

Underlying `iced` widgets is an [Event Loop](https://en.wikipedia.org/wiki/Event_loop), whose role is to bring together all the layers of `iced`. Think of this as the 'dispatch' or traffic light of an `iced` GUI - it controls the flow. The 'runtime' is composed of a number of separate components (`Application`, `user_interface` etc), but these details can be omitted for the sake of this explanation.

The runtime's job is to perform initial setup and then perform the following operations in sequence:

1. _Receive_ application events and user-interaction (mouse movement, clicks, keyboard etc)
2. _Deliver_ these events to the widgets, so they can update their internal state to reflect the happenings of the application or the user
3. _Layout_ the widgets. As the relative size and position of widgets may have changed in response to an event, _layout_ resolves the current position and size of every widget.
4. _Draw_ the widgets. Drawing involves updating the screen with what the widgets want to show.
5. _Wait_. In the current implementation, the runtime waits for a user-interaction or application event before doing it all again.

#### The widgets

Widgets communicate integrate with the rest of `iced` (the 'runtime') by implementing the [Widget](https://docs.rs/iced_native/0.2.2/iced_native/widget/trait.Widget.html) trait. This trait defines:

 * A way to communicate the size needs of the rendered widget (`width()`, `height()`), and resolve its position (`layout()`)
 * A way to provide a [display list](https://en.wikipedia.org/wiki/Display_list) (a list of drawing commands) to the visual backend (`draw()`)
 * A way to update the internal state of the widget based on user interaction or application defined events (`on_event()`)

Notably absent from this trait is any means to indicate to the underlying runtime that an update needs to occur at some time in the future: This current design assumes all updates are the result of user interaction or application events.

### Whats missing

Hopefully the background section explains enough that these points are clear:

 * The `iced` event loop ('runtime') only updates/re-draws its widgets in response to a user or application event. It has no mechanism to redraw itself on its own whim.
 * `iced` Widgets cannot communicate to the runtime that an update/re-draw needs to happen at some point in the future - no such interface exists.
 * Even if the above was not an issue, the `TextInput` only updates in response to user interaction, and has no concept of time, let alone a sense of 'blink me 500ms after the last interaction'.

### Generalizing the problem

Even though this RFC is written to support a blinking text-input, the more general case of a widget wanting to animate itself is a common use-case. We can generalize this problem to that of _periodic animation_ - providing the ability for widgets to update their appearance based on the passage of time. More details on different animation use-cases, and ideas around implementing this can be found in #31 / https://github.com/hecrj/iced/issues/31#issuecomment-703176958.

### Addressing whats missing

#### 1: The event loop

The event loop needs to be able perform an update cycle (ie: handle events if any, update layout if needed, and redraw) when a widget needs it. In the case of a blinking text input cursor, this would be every 500ms, to show/hide the input cursor.

For periodic updates such as these, the behavior of the _Wait_ state needs to be changed. Instead of waiting till the next application event or user interaction, it needs to wait till either the next time a widget needs to be updated, or on the next UI/application event, whichever is first.

#### 2: The widget interface

Widgets need to be able to communicate their need for a update/draw cycle at some time in the future to the event loop, as ultimately the event loop is responsible for updating that cycle.

This necessitates a change to the interface between widgets and the runtime: the [Widget](https://docs.rs/iced_native/0.2.2/iced_native/widget/trait.Widget.html) trait.

The exact change to enable this communication is a matter of API design. However, any change which communicates the soonest moment a widget would need to update would work.

#### 3: The `TextInput` widget

The logic of the widget would need to be updated to:

1. Track the time of the user interaction which affected the cursor position.
2. Indicate to the runtime that an update is needed in 500ms increments after the last user interaction
3. Conditionally emit display list commands to blink the cursor: omitting cursor primitives during the 'hide' phase of the animation (1/2 second), and including cursor primitives during the 'show' phase of the animation (other 1/2 second).

# Reference-level explanation

### TL;DR

 * The problem of a blinking cursor input can be generalized to _periodic animation_ support.
 * The `iced` runtime only performs an update/draw cycle in response to application events and user interaction, but to support periodic animation, we need the runtime to perform a cycle whenever a widget needs it.
 * The current interface (the [Widget](https://docs.rs/iced_native/0.2.2/iced_native/widget/trait.Widget.html) trait) between the runtime and widgets has no means to indicate that a widget is animating, and hence requires a re-draw at sometime in the future.

_The numbering of these points aligns to that of the guided explanation in the previous section_.

### 1: Event loop changes

The event loop needs to be changed to track the soonest moment which an update needs to occur, and to perform an update/draw cycle at this moment.

The intermediate state of a user interface (at the transition of a update/draw cycle) is stored in the `State` struct, from `native/src/program/state.rs`. A new field, `next_animation_draw: AnimationState` can be added to track the next draw required by widgets. More on the `AnimationState` type later, but just know that this type encapsulates the animation requirements of widgets, and this value can be obtained by calling a method on the root widget.

Modifying the behavior of the wait state is surprisingly trivial, thanks to the underlying use of `winit`. Instead of setting the event loops' `control_flow` to `Wait`, we just set it to `WaitUntil( <time of soonest update> )`. This changes the behavior to wait until the next event, or the provided time (whichever is sooner).

As the event loop only updates/draws in response to events/user-interaction, we emit a new synthetic event `AnimationTick` to drive the update cycle.

### 2: Widget trait changes

An update is needed to the [Widget](https://docs.rs/iced_native/0.2.2/iced_native/widget/trait.Widget.html) trait to communicate the animation requirements of a widget - in our case, that the `TextInput` widget needs to be re-drawn in 500ms.

We propose adding a new method to the trait, with a default implementation that just indicate no animation is taking place:

```rust
  fn next_animation(&self) -> AnimationState {
    AnimationState::NotAnimating
  }
```

Widgets that need to animate (such as our `TextInput`) can implement this method to indicate an animation is required:

```rust
  fn next_animation(&self) -> AnimationState {
    AnimationState::AnimateIn(std::time::Instant::now().checked_add(Duration::from_millis(500)))
  }
```

Remaining consistent with the stateless and intuitive feel of the widget trait, `next_animation` is called as part of the update loop, to get the latest set of animation requirements.

### 3: `TextInput` changes

The widget needs to keep track of the last time a user interacted with the input. We can do this by adding a new field to an internal state type `Cursor`:

```rust
pub struct Cursor {
    state: State,
    updated_at: Instant, // new field
}
```

And setting its value when the cursor state is updated:

```rust
pub(crate) fn move_to(&mut self, position: usize) {
    self.updated_at = Instant::now(); // add this line
    self.state = State::Index(position);
}
```

As a corner case, we also need to detect when the input is clicked and gains focus, which we can do by setting `updated_at` during `on_event` when we detect that condition.

### The `AnimationState` type

Widgets need to symbolize their animation requirements to the runtime. We propose creating a new enum type to represent this:

```rust
pub enum AnimationState {
    NotAnimating, // The widget does not need to animate itself, and will only change in response to events.
    AnimateIn(std::time::Instant), // The widget needs to animate itself no sooner than the provided moment.
}
```

A new type seems ideal for the following reasons:

1. It can be easily extended to support future animation use-cases (for instance, continuous/every-frame animations)
2. The runtime only needs to keep track of the _soonest_ it needs to update/draw the widgets. As such, `AnimationState` can implement `std::cmp::Ord` to provide the soonest animation time through `min()`. This will greatly simplify implementation because widgets which contain widgets need only return the `min()` of their contained widgets `AnimationState` values, in response to a `next_animation` call.
3. Existing types arent self describing: In theory, we could get away with an `Option<std::time::Instant>`, but the intent is less obvious then an aptly-named enum.

# Drawbacks

 * An additional method on the widget trait (`next_animation()`) increases the runtime complexity of the update loop. Further, even though the method has a default implementation, its one more thing to think about when implementing a widget.
 * Due to the lack of incremental drawing/update in `iced`, _all_ widgets will be updated & drawn whenever an animation is needed.

# Alternatives

 * Implement a persistent widget tree, and track animation requirements as a persistent property of each widget (#553)
   * Advantages: Could avoid updating/drawing all widgets for an animation cycle, such a widget tree is necessary for other features like accessibility integration
   * Disadvantages: Massively complex, sudden and big change from existing architecture, proposal not yet fully fleshed out or understood
 * Use `on_event` and `messages` to signal to the runtime that a redraw is needed
    * Advantages: Seems somewhat straightforward
    * Disadvantages: `Message` is an associated type, so it will either need to be wrapped somehow or converted into a Tuple. Also, the message-based approach seems harder to reason about than a method that signals current animation requirements.
  * Plumb some indication of animation requirements into a parameter passed to `draw()`
    * Advantages: Straightforward, despite being tedious
    * Disadvantages: Pollutes the responsibilities of `draw()` and makes it no longer free of side-effects

# Future possibilities

 * Motion (frame-by-frame) animations can be added by extending `AnimationState` and associated handling in the event loop.



Please let me know what you think. Thanks! =D

