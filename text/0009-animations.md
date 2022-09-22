# Animations

## Summary

This proposal attempts to introduce state change animations to iced. State change here being defined as any developer configurable values for a widget. An animation on a widget should be as simple as possible for the developer to add. The iterations and redrawing of animations should be handled without requiring messages to be passed to the application. The goal is to be similar to CSS animations and/or [elm-simple-animation](https://github.com/andrewMacmurray/elm-simple-animation). While it is not a goal of this RFC, an implementation requirement is that widgets must be able to signal the window needs a redraw, meaning that animating GIFs, blinking cursors, and even a video widget will be possible.

## Motivation

Developers generally expect a moderntool kit to support animations. This will also close [#31](https://github.com/iced-rs/iced/issues/31).


## Guide-level explanation

The implementation for a developer is rather simple, and backwards compatible. An application can stay exactly the same before and after this RFC is built, and animations can be added at any point in the development cycle. To use Iced, developers must already be familiar with the rust builder pattern to build a `view` in iced. Iced builds widgets block by block. For example, adding `.width(Length::Units(50))` or any other function the widget may offer. This extends the builder pattern to have the option for a widget to add an animation block to the widget. Just like any other widget's builder block, animations are not required to make a valid widget, and exactly what is animatale or anything at all is specific to each widget.

For example a non-animatable button would work like:
```rust
button("Decrement").on_press(Message::DecrementPressed).width(Length::Units(100)),
```

but an animatable widget would just be small change.
```rust
button("Decrement").on_press(Message::DecrementPressed).animate_width(Length::Units(10), Length::Units(100), 1000, Ease::Linear)
```

With the current implementation an animation goes **from** `Length::Units(10)` **to** `Length::Units(100)` for a **duration** of `1000`ms, with a linear **ease**.

#### Ease
In the example above **ease** imapacts the rate that the animation runs. This concept is not new for animations in other toolkits, but is new to Iced. The [Mozilla CSS Documentations](https://developer.mozilla.org/en-US/docs/Web/CSS/easing-function) explains this concept very well. Iced should eventually support all of the different ease versions as CSS.
We could also add a `Ease::loop` or `Ease::bounce` etc for infinite animations, but this is likely to change, see `Unanswered Questions` below.

Widget developers will also need to be familiar with the concept of a `step` if they wish for their widget to animate, See `Implementation strategy` below for details.

## Implementation strategy

Iced is retained mode, but widgets don't live for long. Every time Iced renders the UI nearly everything is rebuilt. Iced just only does that when there is an event, rather than every frame like an immediate mode toolkit would. Iced’s widget dimensions and theme are ephemeral currently. When a widget is defined with `.width(Length::Fill)`, that isn't saved until the next time the layout is built. That data is only stored in the widget struct,
```rust
pub struct Button<'a, Message, Renderer>
where
    Renderer: crate::Renderer,
    Renderer::Theme: StyleSheet,
{
    content: Element<'a, Message, Renderer>,
    on_press: Option<Message>,
    width: Length,
    height: Length,
    padding: Padding,
    style: <Renderer::Theme as StyleSheet>::Style,
}
```
but that is different than the widget `state`
```rust
pub struct State {
    is_pressed: bool,
}
```
which also doesn't live very long. Though the main difference is the tree of widget states becomes a [`cache`](https://docs.iced.rs/iced_native/user_interface/struct.Cache.html) that is available for the next time iced redraws the GUI. The intention of the cache is to speed up building the widget tree. This RFC adds another job of the cache, it will now also be the tool to "pass the baton" of the current animation state to the next time the [UserInterface](https://docs.iced.rs/iced_native/user_interface/index.html) is built.

That "baton" is the `Animation` type
```rust
pub enum Animation {
	Idle(...)
	Working(...)
}
```
An animation is either `Idle` or `Working`, `Idle` means that iced doesn't have to do anything with the animation, it is either done, or never required to animate at all. A `Working` animation tells iced that this animation needs to be redrawn, that is done by the new concept of a step:

```rust
fn step_state(&mut self, state: &mut tree::State, time: Instant) {}
```
Which is a function in the [Widget trait](https://docs.iced.rs/iced_native/widget/trait.Widget.html). It has a default implementation of `{}`, so that widgets don't have to animate if they don't want to. Making this backwards compatible with older widgets, and simpler to implement custom widgets.
Before we get back into the implementation what is the concept of a `step`?
A step is the process of interpolating a widget's state through an animation. Iced will call `step_state` on each widget that has a state just before redrawing the GUI. The animatable state will have to calculate what value it should have using the previous state, ease function, and the new time. After the animations have been stepped the new value can be used by Iced for the `Layout` and `Draw`.

Though for most widgets `step`ping the state will be very similar to
```rust
    fn step_state(&mut self, state: &mut tree::State, time: usize) {
        state.downcast_mut::<State>().step(time)
    }
```
in the `Widget` implementation for the specific widget, then
```rust
    pub fn step(&mut self, now: Instant) {
        self.width = self.width.step(now);
        self.height = self.height.step(now);
    }
```
in the widget's state `impl`, because the `Animation` type can implement a step function for each type of Animation.

The only quirk is that in the `layout` function for a widget must then use the state for it's layout calculations. For example, a `button`'s implementation would need to be:
```rust
    fn layout(
        &self,
        renderer: &Renderer,
        limits: &layout::Limits,
        tree: &Tree,
    ) -> layout::Node {
        let (width, height) = match &tree.state {
            crate::widget::tree::State::Some(s) => {
                let btn_state = tree.state.downcast_ref::<State>();
                (btn_state.width.at(), btn_state.height.at())
            }
            _ => (self.width.at(), self.height.at())
        };
        layout(
            renderer,
            limits,
            width,
            height,
            self.padding,
            |renderer, limits| {
                self.content.as_widget().layout(renderer, limits, tree)
            },
        )
    }
```
where `.at()` is a convenience function that extracts the current animation state for both `Idle` animations and `Working` animations.

#### Diff
Iced calls the step where the previous layout's widget tree is compared against the new `view`, `diff`ing. This is where the "baton" handoff happens and the `step` function are called. The isssue here is that Iced's current `diff` functions use references to data. `step`ping an animation requires modifying data, so the `diff` state needs to be mutable. This isn't  an issue as `UserInterface::build` already owns the data, it just means that Iced needs to have `diff` and `diff_mut`. The other effected functions are `diff_children_mut`, and `diff_children_custom_mut`. Though for a vast majority of developers they wont need to touch these functions.

#### Integrating iced (this is still in progress)
Custom integrations as well as Iced's `winit` and `glutin` will need to handle the diff state differently. Because `step`ping requires the time to calculate the new animation state all three will have to hold an [`Instant`](https://doc.rust-lang.org/std/time/struct.Instant.html) that is passed to the diff state for each loop. As well currently Iced's `diff` function doesn't return anything. A widget that needs reanimation will return that it needs reanimation, so the iced runtime, or custom integrations will know that they need to build the `UserInterface` again.
The return type will be of `RequestAnimationFrame` or `Duration(time)`. `RequestAnimationFrame` should be as fast as the user's display, or as fast as the custom integration would like, this is for smooth animations like sliding, resizing, or color changes. `Duration` is for things like cursor blink animations or GIFs or videos, as they would like to be redrawn at their frame rate, which is in some `Duration`.


## Drawbacks

Animations will require more memory as data to calculate the next frame are required, not just the current state. As well their will be minor performance loss as the `step` will have to check each animatable state even if the animation is `Idle`.


## Rationale and alternatives

[anim-rs](https://github.com/joylei/anim-rs) style animations. Anim-rs requires that the user holds the animations state, timeline, and adds a subscription, but it does have the benefit of making basically anything animatable without having to modify Iced itself.

## Unresolved questions

- The diff accumulator, but that isn't' implemented yet.
- Theme animations, should be similar, but I haven't looked at if/how we can transition a button from one theme option to another.
- How many widgets should be animatable? As stated in the drawbacks there is a minor performance loss be having to check the animations state each `step`. This could be avoided by only having dimension animations on `column`s, `row`s, `space`s and `container`s, then setting the children to `Length::Fill`, but that is (1) less ergonomic, (2) not proven if creating more widgets would be more performant that running more `match` statements.
- Chaining animations. Some animations need to do something **then** another animation. My current implementation only animates one value then stops. This should still be possible with most of this RFC, but the API would have to be changed to be adding a `.animation()` function that takes a two-dimensional `vec` of animations. For example something that grows and slides would need `.animate(vec![vec![slide_down, slide_right], vec![set_height_smaller]])`
- Building off chained animations, that would break the proposed `Ease::Loop` above, it would need to be `.animate()`, `.animate_loop()`, `animate_bounce()`, etc.


## [Optional] Future possibilities

-  This RFC makes no attempt to add blinking cursors, nor animated GIFs, nor a video widget. Just the framework for those.
