# Theming

## Summary

This proposal introduces the first-class concept of a __theme__ in iced. A __theme__ is defined as some data type capable of changing the default look of the widgets of an application.


## Motivation

Applications tend to use a single, consistent look for all of its widgets.

However, styling in iced currently works in a case-by-case basis. In other words, the style of every widget in an application needs to be explicitly set with the `style` methods of the specific widget. For instance:

```rust
let text_input = TextInput::new(/* ... */).style(theme);
let button = Button::new(/* ... */).style(theme);
let slider = Slider::new(/* ... */).style(theme);
let progress_bar = ProgressBar::new(/* ... */).style(theme);

let scrollable = Scrollable::new(/* ... */)
    .style(theme)
    .push(text_input)
    .push(button);
    .push(slider);
```

In the snippet above, all of the widgets use the same `theme` as their `style`. The only way to achieve a consistent look in an application is by passing the same `theme` to every widget. Obviously, this is cumbersome as well as error-prone.

Given how common and basic this use case is, iced should introduce new ideas to support a single, centralized, consistent theme for an application.

## Guide-level explanation

### First-class themes

An `Application` has a new `Theme` associated type alongside a new `theme` method:

```rust
struct Example;

impl Application for Example {
    type Theme = /* ... */;

    // ...

    fn theme(&self) -> Self::Theme {
        // ...
    }
}
```

The `theme` method works analogously to other methods in the trait like `view`, `title`, `mode`, etc. The theme returned in this method will be used to draw the `Application`.


Widgets may require the `Application::Theme` to implement their own specific style sheets. For instance, a `Button` requires `Application::Theme` to implement `button::StyleSheet` before it can be used.

Furthermore, the `StyleSheet` trait of a widget can introduce associated types to offer additional styling flexibility. For instance, `button::StyleSheet` introduces a `Style` associated type which can be used to set the specific style of a `Button` instance with the `Button::style` method.

Widgets and `Element` in `iced` now have a `Theme` generic type, alongside the potentially present `Message`, that needs to be specified in the type signatures. The compiler will be able to infer the type in other instances, so the overall impact on the API should be relatively low.

Let's see an example of how all of this works together! Let's say we want to use our own custom theme type in our `Application`. We start by defining our custom `Theme` type:

```rust
struct Theme {
    text: Color,
    action: Color,
    positive: Color,
    negative: Color,
}
```

Then, we define the styles of a `Button` we will have in our application:

```rust
#[derive(Debug, Clone, Copy)]
enum Button {
    Primary,
    Positive,
    Destructive,
}

// The `Button` widget demands that the `Style` of a `button::StyleSheet`
// implements `Default`
impl Default for Button {
    fn default() -> Self {
        Self::Primary
    }
}
```

Now, we can implement `button::StyleSheet` for our `Theme`:

```rust
use iced::button;

impl button::StyleSheet for Theme {
    type Style = Button;

    fn active(&self, style: Button) -> button::Style {
        // We can use the `Theme` colors in `self` here and produce a different
        // `button::Style` for each `Button` style
        match style {
            Button::Primary => /* ... */,
            Button::Positive => /* ... */,
            Button::Destructive => /* ... */,
        }
    }
}
```

Then, we can use our brand new custom `Theme` in our `Application`:

```rust
use iced::pure::{Application, Element};
use iced::pure::{button, column};

use theme::{self, Theme};

struct Example;

impl Application for Example {
    type Theme = Theme;

    // ...

    // Notice that `Element` needs to know about the custom `Theme` now!
    fn view(&self) -> Element<Self::Message, Self::Theme> {
        // Let's show all of our button styles
        column()
            .push(button("Primary")) // The default `Button` style is used!
            .push(button("Positive").style(theme::Button::Positive))
            .push(button("Destructive").style(theme::Button::Destructive))
            .into()
    }

    fn theme(&self) -> Theme {
        // We generate the `Theme` on the fly here, but we could also store it
        // in our application state and return a copy
        Theme {
            /* ... */
        }
    }
}
```

As you can see, the new `Theme` associated type ties everything up together in a type-safe way. If we were to change our `Theme` to a different type, our `view` code would stop compiling unless the new type supported the same button styles.


### Built-in themes
Overall, custom themes are considered an advanced use case.

Users that are getting started with iced do not want to be forced to build a custom theme and manually implement the `StyleSheet` trait of every widget they need. Additionally, they may not be interested in fine-tuning the looks of the application, but instead they may just want to choose an existing good-looking theme.

For these reasons, `iced` provides a built-in `Theme` type and `theme` module with styles for some widgets that can be used out of the box:

```rust
use iced::pure::{Application, Element};
use iced::pure::button;
use iced::theme::{self, Theme};

struct Example;

impl Application for Example {
    type Theme = Theme;

    fn view(&self) -> Element<Message> {
        button("Hello!").style(theme::Button::Secondary)
    }

    fn theme(&self) -> Theme {
        Theme::Light
    }
}
```

Notice how, in this case, we didn't specify the `Theme` in the `Element`. All of the built-in widgets (and `Element`) will use the built-in `Theme` type as the default type for their new `Theme` generic type. As a result, when using the built-in `Theme`, all of the type signatures can stay simple!

For now, the built-in `Theme` will be a simple enum with three variants:

```rust
pub enum Theme {
    Light,
    Dark,
    Custom(Palette),
}
```

The built-in `Theme`, as well as the supported widget variants, can be thoroughly extended in the future! See "[Future possibilities](#future-possibilities)".

The `Custom` variant can be used to define a custom color `Palette`, which is defined as follows:

```rust
pub struct Palette {
    background: Color,
    text: Color,
    primary: Color,
    success: Color,
    danger: Color,
}
```

Internally, all of the `Theme` variants will define its own `Palette`. In other words, both `Light` and `Dark` themes are just built-in custom color palettes:

```rust
assert_eq!(Theme::Light.palette(), Theme::Custom(Theme::Light.palette()).palette());
```

### Extending the built-in themes

The built-in `Theme` can be supported by custom widgets defined in other crates of the ecosystem.

A custom widget can define its own `StyleSheet` trait:

```rust
pub trait StyleSheet {
    type Style: Default + Copy;

    // ...
}
```

And then, implement the trait for the built-in `Theme` in the same crate:

```rust
use iced::Theme;

impl StyleSheet for Theme {
    type Style = /* ... */;

    // ...
}
```

`Theme` exposes an `extended_palette` method that can be leveraged in the `StyleSheet` implementation to choose the proper colors. `extended_palette` returns a `palette::Extended` type that contains different shades generated from the original `Palette` of a `Theme`.

These internal details are likely to change as we fine-tune the specific styling for `iced`, which falls out of scope of this RFC.

### `Sandbox` and simplicity

For simplicity, the `Sandbox` trait does not have the `Theme` associated type and, as a result, will always use the default built-in `Theme` type.

As a consequence, users will need to migrate to the `Application` trait if they decide to leverage a custom `Theme` type. However, `Sandbox` does have a `theme` method that can be used to change the `Theme` variant.


## Implementation strategy

We introduce a `Theme` associated type to the `Renderer` trait in `iced_native`.

As a consequence, widgets can easily add bounds on this `Theme` as needed. For instance, the `Button` described above can be implemented as follows:

```rust
pub use iced_style::button::{Style, StyleSheet};

pub struct Button<'a, Message, Renderer>
where
    Renderer: iced_native::Renderer,
    // The `Theme` needs to implement `StyleSheet`
    Renderer::Theme: StyleSheet,
{
    // ...
    // A `Button` stores the `Style` defined by the `StyleSheet`
    style: <Renderer::Theme as StyleSheet>::Style,
}

impl<'a, Message, Renderer> Button<'a, Message, Renderer>
where
    Renderer: iced_native::Renderer,
    Renderer::Theme: StyleSheet,
{
    pub fn new(content: impl Into<Element<'a, Message, Renderer>>) -> Self {
        Button {
            // ...
            style: Default::default(),
        }
    }

    // ...

    /// Sets the style of this [`Button`].
    pub fn style(
        mut self,
        style: impl Into<<Renderer::Theme as StyleSheet>::Style>,
    ) -> Self {
        self.style = style.into();
        self
    }
}
```

Additionally, we need to provide the current `Renderer::Theme` to the widgets during `draw`. Therefore, we need to introduce the `Renderer::Theme` as an argument to the `draw` method in the `Widget`:

```rust
pub trait Widget<Message, Renderer>
where
    Renderer: crate::Renderer,
{
    /// Draws the [`Widget`] using the associated `Renderer`.
    fn draw(
        &self,
        renderer: &mut Renderer,
        theme: &Renderer::Theme,
        style: &renderer::Style,
        layout: Layout<'_>,
        cursor_position: Point,
        viewport: &Rectangle,
    );
}
```

Notice that we need to add the `crate::Renderer` bound to the `Renderer` generic type in order to properly reference `Renderer::Theme` in the trait body.

Besides the `Widget` trait, the `Overlay` trait needs to change analogously, as well as their `pure` counterparts.

Now that widgets need a `Renderer::Theme`, we will need to provide it to `UserInterface::draw` as well:

```rust
impl<'a, Message, Renderer> UserInterface<'a, Message, Renderer>
where
    Renderer: crate::Renderer,
{
    pub fn draw(
        &mut self,
        renderer: &mut Renderer,
        theme: &Renderer::Theme,
        cursor_position: Point,
    ) -> mouse::Interaction {
        // ...
    }
}
```

The `update` method in `program::State` in `iced_native` will need it too:

```rust
impl<P> State<P>
where
    P: Program + 'static,
{
    pub fn update(
        &mut self,
        bounds: Size,
        cursor_position: Point,
        renderer: &mut P::Renderer,
        theme: &<P::Renderer as crate::Renderer>::Theme,
        clipboard: &mut dyn Clipboard,
        debug: &mut Debug,
    ) -> Option<Command<P::Message>> {
        // ...
    }
}
```

Since we want `Application` to choose the `Theme`, we will need to make the `Renderer` in `iced_graphics` generic over a `Theme` generic type:

```rust
#[derive(Debug)]
pub struct Renderer<B: Backend, Theme> {
    backend: B,
    primitives: Vec<Primitive>,
    theme: PhantomData<Theme>,
}
```

And then implement `iced_native::Renderer` as follows:

```rust
impl<B, T> iced_native::Renderer for Renderer<B, T>
where
    B: Backend,
{
    type Theme = T;

    // ...
}
```

These changes prompt the `Renderer` in `iced_wgpu` and `iced_glow` to change analogously:

```rust
pub type Renderer<Theme = iced_native::Theme> =
    iced_graphics::Renderer<Backend, Theme>;
```

Notice that we set the default type of the `Theme` generic type as `iced_native::Theme`, which should reduce verbosity when choosing the built-in `Theme`.

The `Compositor` in `iced_wgpu` and `iced_glow` need to also be aware of the `Theme` too:

```rust
pub struct Compositor<Theme> {
    // ...
    theme: PhantomData<Theme>,
}
```

So they can implement the `Compositor` traits from `iced_graphics`:

```rust
impl<Theme> iced_graphics::window::Compositor for Compositor<Theme> {
    type Renderer = Renderer<Theme>;

    // ...
}
```

Now, we can introduce the new `theme` method to the `Application` trait in `iced_winit`:


```rust
trait Application: Program {
    // ...

    /// Returns the current [`Theme`] of the [`Application`].
    fn theme(&self) -> <Self::Renderer as iced_native::Renderer>::Theme;

    // ...
}
```

We initialize the `theme` in `run_instance`:

```rust
// ...

let mut state = State::new(&application, &window);
let mut viewport_version = state.viewport_version();
let mut theme = application.theme();

// ...
```

Then, we properly update it after an `update`:

```rust
// ...

theme = application.theme();
user_interface = ManuallyDrop::new(build_user_interface(
    &mut application,
    cache,
    &mut renderer,
    state.logical_size(),
    &mut debug,
));

// ...
```

And we provide it to `UserInterface::draw`:

```rust
debug.draw_started();
let new_mouse_interaction = user_interface.draw(
    &mut renderer,
    &theme,
    state.cursor_position(),
);
debug.draw_finished();
```

`iced_glutin` needs to change analogously.

Finally, we can update the `Application` trait in the `iced` crate. We need to introduce the `Theme` associated type and the `theme` method:

```rust
pub trait Application: Sized {
    // ...

    /// The theme of your [`Application`].
    type Theme;

    // ...

    /// Returns the current [`Theme`] of the [`Application`].
    fn theme(&self) -> Self::Theme;

    // ...
}
```

Then, we complete its implementation of both `iced_winit::Program` and `iced_winit::Application` traits:

```rust

impl<A> iced_winit::Program for Instance<A>
where
    A: Application,
{
    type Renderer = crate::renderer::Renderer<A::Theme>;

    // ...
}

impl<A> crate::runtime::Application for Instance<A>
where
    A: Application,
{
    fn theme(&self) -> A::Theme {
        self.0.theme()
    }
}
```

We also need to introduce the `Theme` generic type to all the type aliases for widget (and `Element`):

```rust
pub type Column<'a, Message, Theme = crate::Theme> =
    iced_native::widget::Column<'a, Message, crate::Renderer<Theme>>;

pub type Row<'a, Message, Theme = crate::Theme> =
    iced_native::widget::Row<'a, Message, crate::Renderer<Theme>>;

// ...

pub type Element<'a, Message, Theme = crate::Theme> =
    crate::runtime::Element<'a, Message, crate::Renderer<Theme>>;
```

As before, we set the default type of to the built-in `Theme`.

And that should be it!


## Drawbacks

The design introduces some complexity on the type signature of the widgets (and `Element`) when using custom themes. However, this issue isn't particularly worrying because users can always define their own type aliases. For instance:

```rust
pub type Element<'a, Message> = iced::Element<'a, Message, MyCustomTheme>;
```

This could be a nice side-effect, since encouraging users to build their own `view` helpers on top of `iced` is great.

An important drawback is that the current design does not allow for widget customization outside of a `Theme`. For instance, if a user is using an existing `Theme`, they will not be able to easily fine-tune the specific properties of the styling of a `Button` to their liking. They will be forced to create a new custom theme.

However, I believe this is a good thing since consistency is the main motivation for this feature. Furthermore, users can always leverage composition to reuse existing themes.


## Rationale and alternatives
This design is quite elegant, simple, and type-safe.

Using the existing `Renderer` generic type in the widgets to introduce bounds for the `Theme` is quite straightforward and powerful.

Furthermore, the design leverages the type system and generics to monomorphize the supported variants of a `Theme`. As a result, it removes the `Box<dyn StyleSheet>` present everywhere in the current widgets, which should in turn help reduce allocations in `view` code.

The design also allows for easy customization and extensibility. Supporting custom themes is the main satisfied use case and should allow the community to create and maintain their own custom theme crates. The built-in `Theme` can also be easily extended as well.


## Unresolved questions

- All questions resolved for now!


## Future possibilities

- In the future, the built-in `Theme` can be extended with additional variants for supporting different color schemes.
- An `Autodetect` variant could be added to the built-in `Theme` to let `iced` choose the best fitting variant of the `Theme` based on the environment (e.g. OS settings).
