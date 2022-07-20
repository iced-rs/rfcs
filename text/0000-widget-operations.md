# Widget Operations

## Summary

This proposal introduces the idea of __widget operations__ in iced. A __widget operation__ is defined as some logic that can traverse (and operate on) the widget tree of an iced application in order to query or update some widget state.


## Motivation

The non-pure widget API in iced forces users to explicitly store widget state in their application state. This approach, while cumbersome, has a nice benefit. Given that all of the widget state is explicitly defined in the application state, it's possible for users to easily modify the internal state of a specific widget during an `update`.

For instance, if we have a `TextInput` and a `Button` in our application:

```rust
struct Example {
    input: text_input::State,
    button: button::State,
}
```

We can easily change the state of the `input` in `update`. For example, let's say we want to focus the `input` after the `button` is pressed. We can just write:

```rust
fn update(&mut self, message: Message) {
    match message {
        Message::ButtonPressed => {
            self.input = text_input::State::focused();
        }
        // ...
    }
}
```

Since we cannot expect users to manage and synchronize all of the widget internal state through The Elm Architecture, this is a very common and perfectly valid use case.

In [#1284](https://github.com/iced-rs/iced/pull/1284), a new widget API was introduced with the goal to replace the current non-pure approach. And precisely, the main difference is the lack of explicit widget state management.

In practice, this means that users do not have to store widget state in their application state anymore. But, as a consequence, users cannot change the internal state of a widget during an `update`, since there is no widget state to refer to.

For instance, when using the pure widget API, the previous `Example` struct would not hold any widget state:

```rust
struct Example;
```

Thus, it would not be possible to refer to the state of the `TextInput`:

```rust
fn update(&mut self, message: Message) {
    match message {
        Message::ButtonPressed => {
            // How do we focus the `TextInput` here?
        }
        // ...
    }
}
```

If we want the new pure API to replace the current one, we need to introduce new ideas in iced that can be used to query and update internal widget state.


## Guide-level explanation

There are only two new ideas that end-users need to get familiarized with: **operations** and **identifiers**.

### Operations
Widget modules can export **widget operations** in their public interface. A widget operation is a `Command` and, as a result, it can be run by any implementor of the `Application` trait.

For instance, we could satisfy the previous use case if the `text_input` module exposed a `focus` operation:

```rust
fn update(&mut self, message: Message) -> Command<Message> {
    match message {
        Message::ButtonPressed => {
            // We build the `focus` operation and return the resulting `Command`
            text_input::focus(MY_TEXT_INPUT_ID)
        }
        // ...
    }
}
```

In the example above, `focus` is a function exported by the `text_input` module that can be used to build a widget operation that focuses a specific `TextInput`. Its output is a `Command` that any `Application` can run.

Since we may produce many different text inputs in our `Application::view`, we need some way to **identify** the specific text input. As a consequence, some widget operations may need **identifiers**.


### Identifiers

A **widget identifier** is an instance of some opaque type `Id` exported by a widget module. A widget will only support identifiers if it exposes an operation that needs them.

For instance, the `MY_TEXT_INPUT_ID` constant in the previous example could be defined as follows:

```rust
const MY_TEXT_INPUT_ID: text_input::Id = text_input::Id::unique();
```

For any built-in widget, the `unique` constructor of an `Id` type always produces a different instance every time it's called:

```rust
use some_widget::Id;

let some_id = Id::unique();
let another_id = Id::unique();

assert_ne!(some_id, another_id);
```

Any built-in `Id` type also generally offers a simple `new` constructor that takes a `String`, which can be used in `view` code to generate identifiers from an external source of data at runtime (as opposed to using `unique` at compile-time).

A widget that supports identifiers has an `id` method that can be used to set the `Id` in `view` code, effectively connecting `view` logic with any operation produced in `update`.

For instance, if we include the following `TextInput` in our `view`:

```rust
text_input("Some placeholder", some_value, Message::InputChanged)
    .id(MY_TEXT_INPUT_ID)
```

Then, returning the `Command` produced by `text_input::focus(MY_TEXT_INPUT_ID)` in `Application::update` will cause this specific `TextInput` to gain focus.


## Implementation strategy

### A new kind of `Command`
A new `widget` method will be introduced to `Command` in `iced_native`:

```rust
impl<T> Command<T> {
    pub fn widget(operation: impl widget::Operation<Output = T>) -> Self {
        Self::single(Action::Widget(Box::new(operation)))
    }
}
```

As shown above, the implementation of `Command::widget` will use a new `command::Action`:

```rust
pub enum Action<T> {
    // ...
    /// Run a widget operation.
    Widget(Box<dyn widget::Operation<Output = T>>)
}
```

The new `Command::widget` method will allow widget developers to expose a widget operation as a `Command`. For instance, we could write `text_input::focus` as follows:

```rust
use iced_native::command::{self, Command};

pub fn focus<Message>(id: Id) -> Command<Message> {
    Command::widget(Focus(id))
}
```

As shown above, `Command::widget` takes an implementor of a brand new trait: `widget::Operation`.

### The `Operation` trait
The `Operation` trait in the `widget` module defines the logic of a widget operation. A widget operation usually represents some state that is changed as it traverses a widget tree, producing some output `T` as a result.

An operation is split into different methods that represent different widget types. Widgets will call the applicable methods of the `Operation` trait given their particular nature. An operation may be called multiple times per widget, if applicable.

In a way, the `Operation` trait is analogous to [the `Hasher` trait](https://doc.rust-lang.org/std/hash/trait.Hasher.html), but the hashed values (the operands!) are the widgets themselves.

The trait is defined as follows:

```rust
use crate::widget::Id;
use crate::widget::state;

pub trait Operation<T> {
    fn clickable(&mut self, state: &mut dyn state::Clickable, id: Option<Id>);
    fn focusable(&mut self, state: &mut dyn state::Focusable, id: Option<Id>);
    fn editable(&mut self, state: &mut dyn state::Editable, id: Option<Id>);
    fn text(&mut self, contents: &str, id: Option<Id>);

    fn container(
        &mut self,
        _id: Option<Id>,
        operate_on_children: &dyn FnOnce(&mut dyn Operation<Output = Self::Output>),
    );

    // ...
    // Further methods can be added for additional widget types!
    // ...

    fn finish(self) -> Outcome<T>;
}
```

As shown, an operation can operate on different types of widget. Where applicable, the methods of the trait receive mutable widget state that can be used to query it or update it.

For instance, we could implement the `focus` operation by leveraging the `focusable` method:

```rust
use iced_native::command::{self, Command};
use iced_native::widget;
use iced_native::widget::state;

pub fn focus<Message>(id: Id) -> Command<Message> {
    Command::widget(Focus(id))
}

struct Focus(Id);

impl<T> widget::Operation<T> for Focus {
    fn focusable(&mut self, state: &mut dyn state::Focusable, id: Option<Id>) {
        // If the current widget has an identifier...
        if let Some(candidate_id) = id {
            // And it matches the identifier we are looking for...
            if self.0 == id {
                // Then, we focus the input!
                state.focus();
            }
        }
    }

    fn container(
        &mut self,
        _id: Option<Id>,
        operate_on_children: &dyn FnOnce(&mut dyn Operation<Output = Self::Output>),
    ) {
        // If the current widget is a container, we just keep traversing the tree.
        operate_on_children(self)
    }

    fn finish(self) -> Outcome<T> {
        Outcome::None
    }

    // ...
}
```

In the `focusable` method we look for any focusable widget that has the identifier we are interested in, then we use the generic `Focusable` widget state to `focus` that specific widget.

The `container` implementation will be called by any widget that contains other widgets and, as a result, it can be leveraged to control the traversal of the widget tree. In this case, we just keep traversing the widget tree unconditionally.

An `Operation` may produce some `Outcome` when `finish` is called. The `focus` operation does not produce any.

The implementation of the other methods in this particular example is empty and has been omitted. In fact, the `Operation` trait provides a default implementation for all of its methods except `container`:

```rust
pub trait Operation<T> {
    fn container(
        &mut self,
        _id: Option<Id>,
        operate_on_children: &dyn FnOnce(&mut dyn Operation<Output = Self::Output>),
    );

    fn clickable(&mut self, state: &mut dyn state::Clickable, id: Option<Id>) {}
    fn focusable(&mut self, state: &mut dyn state::Focusable, id: Option<Id>) {}
    fn editable(&mut self, state: &mut dyn state::Editable, id: Option<Id>) {}
    fn text(&mut self, contents: &str, id: Option<Id>) {}

    // ...

    fn finish(self) -> Outcome<T> {
        Outcome::None
    }
}
```

Therefore, an empty implementation of an `Operation` will be the "identity" operation and will simply traverse the whole widget tree without doing anything.

### Generic widget state

For reusability, widget operations should be decoupled from particular widget implementations. Furthermore, it is necessary for operations to work even when custom widgets are present in a `view`.

Because of this, the `Operation` trait relies on a new set of generic traits meant to represent different kinds of widget state. For instance, instead of directly relying on `text_input::State`, the `Operation` trait takes an implementor of the `state::Focusable` trait:

```rust
pub trait Focusable {
    fn is_focused(&self) -> bool;
    fn focus(&mut self);
    fn unfocus(&mut self);
}
```

This way, an `Operation` can support virtually any widget, as long as the internal widget state implements the proper state traits. For example, `text_input::State` will implement `state::Focusable`:

```rust
impl state::Focusable for State {
    fn is_focused(&self) -> bool {
        State::is_focused(self)
    }

    fn focus(&mut self) {
        State::focus(self)
    }

    fn unfocus(&mut self) {
        State::unfocus(self)
    }
}
```

Further widgets could choose to implement this trait for their internal state, depending on the nature of the widget. It is up to the implementation of a `Widget` to properly support the `Operation` API.


### Outcomes

The `Outcome` of an operation can take one of 3 shapes:

* `None` means the operation produced no result.
* `Some` means the operation produced a result of type `T`. This type `T` will be, at the same time, the output of its respective `Command`. Generally, this will lead to a new `Message` being produced and fed to `update`.
* `Chain` means the operation produced a brand new operation as a result, and should be executed.

```rust
pub enum Outcome<T> {
    None,
    Some(T),
    Chain(Box<dyn Operation<Output = T>>),
}
```

In many cases, traversing the widget tree once may not be enough to complete an operation. For instance, an operation like `text_input::focus_previous` needs to traverse the tree once to find the current focused widget, unfocus it, and then traverse it again to focus the previous focusable widget with the information gathered in the first traversal.

For this reason, it is possible for an operation to be split into many, chained smaller ones.


### Running an operation

The shell implementations (i.e. `iced_winit` and `iced_glutin`, currently) will need to handle the new `Widget` variant of `command::Action`:

```rust
for action in command.actions() {
    match action {
        // ...
        command::Action::Widget(operation) => {
            user_interface.operate(renderer, operation);

            let outcome = operation.finish();

            // Handle the outcome...
        }
    }
}
```

As shown above, `UserInterface` will have a new `operate` method:

```rust
impl<'a, Message, Renderer> UserInterface<'a, Message, Renderer>
where
    Renderer: crate::Renderer,
    Renderer::Theme: application::StyleSheet,
{
    // ...

    /// Applies a [`widget::Operation`] to the [`UserInterface`].
    pub fn operate(
        &mut self,
        renderer: &Renderer,
        operation: &mut dyn widget::Operation<Message>,
    ) {
        self.root.operate(Layout::new(&self.base), operation);

        if let Some(layout) = self.overlay.as_ref() {
            if let Some(overlay) =
                self.root.overlay(Layout::new(&self.base), renderer)
            {
                overlay.operate(Layout::new(layout), operation);
            }
        }
    }
}
```

This new method in turn uses a new `operate` method on both the `Widget` and `Overlay` traits:

```rust
/// Applies an [`Operation`] to the [`Widget`].
fn operate(
    &mut self,
    _state: &mut Tree, // Only the pure traits have this argument
    _layout: Layout<'_>,
    _operation: &mut dyn Operation<Message>,
);
```

Each particular widget implementation will call the proper methods of the `Operation` based on the nature of the widget.

For instance, `TextInput` could implement `operate` as follows:

```rust
impl<'a, Message, Renderer> Widget<Message, Renderer>
    for TextInput<'a, Message, Renderer>
where
    Message: Clone,
    Renderer: text::Renderer,
    Renderer::Theme: StyleSheet,
{
    // ...

    fn operate(
        &self,
        tree: &mut Tree,
        _layout: Layout<'_>,
        operation: &mut dyn widget::Operation<Message>,
    ) {
        let state = tree.state.downcast_mut::<text_input::State>();

        // We can call `focusable` since `text_input::State` implements
        // `state::Focusable`
        operation.focusable(state, self.id);

        // `editable` can be called as well, analogously.
        operation.editable(state, self.id);
    }
}
```

Other widgets will be implemented in a similar way!

## Drawbacks

There aren't many drawbacks to this proposal, since most of the changes described here do not break any existing, end-user-facing APIs. Instead, most of the changes revolve around new ideas without affecting the existing codebase.


## Rationale and alternatives

The design described here is relatively simple. Since operations are exposed through the `Command` API, end users only really need to get familiarized with the idea of identifiers.

Identifiers are common in the Web and other GUI toolkits, and so the friction for end users should be minor.

Furthermore, Elm uses a very similar design where [internal widget state can be updated through tasks and identifiers as well](https://package.elm-lang.org/packages/elm/browser/latest/Browser-Dom#focus).

The design should be generic enough to work well with custom widgets and, at the same time, should be extensible and allow us to iterate easily if new widgets types are necessary.

Finally, the implementation should be quite straightforward with no apparent hacks or important refactors necessary.


## Unresolved questions

- Should we rename `Outcome` to something else? An operation producing a `Result`  makes sense, but a `Result` type in Rust generally has an error variant and an operation cannot really fail as of now.
- What other methods should we include in `widget::Operation`? Since many methods of an operation can be called per widget, should we include methods to mark the "start" and "end" of a widget?


## Future possibilities

- We could allow some methods of the generic state traits to produce messages as output. This would allow us to implement interesting use cases in the future (e.g. a test framework that queries and interacts with the GUI exclusively through operations).
- Many useful operations should be possible with this design and could be built-in in the library, like querying the current size of a particular widget or controlling a `Scrollable` programmatically.
