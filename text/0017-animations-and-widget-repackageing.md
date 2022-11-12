# Animations + Widget Repackaging

## Summary

In Iced each widget has a state. This is data that is widget data that can be handled by Iced without developer intervention. This is mostly for developer convenience, widget repackaging aims to expand on that convenience and allow for simple animations, but also for optimizations where the widget may use a cache. After each render of the UI iced stores this widget state in a cache, to prevent Iced from having to allocate on the stack for each widget each render. This RFC intends on expanding that behavior.
The goals of this RFC are to allow for a simple animation API and widget optimizations like a text render cache via:
	2 ) Widget repackaging: Add a step where data from the state can be used for calculations and/or re-injected into the widget itself.
	3 ) Widget Redraw:      Allow widgets to request redraw.


## Motivation

In a modern toolkit developers expect complicated animations with a rather simple API and efficient widget redraws. While the only plan for the efficient redraws is a text renderer cache, a cache is useful in general and is likely to be needed in other widgets. The issue is trying to satisfy expectations from toolkits that contain side effects into a toolkit with a strong adherence to the Elm architecture.


## Guide-level explanation

Most of this RFC is technical details that will not need to be touched by the end user, but the animations API will be visible to the end user.

Not all widgets will be animatable, nor is it guaranteed that every value is animatable, but assuming we are using an animatable widget the APi will be along the lines of:
```rust
button("I Grow and Shrink!")
    .height(Length::Units(10))
    .width(Length::Units(10))
    .animation(
	animation![
		button::Keyframe::new(Duration::from_secs(3))
		    .height(Length::Units(100))
		    .width(Length::Units(100))
		button::Keyframe::new(Duration::from_secs(6))
		    .height(Length::Units(10))
		    .width(Length::Units(10))
		button::Keyframe::new(Duration::from_secs(9))
		    .height(Length::Units(100))
		    .width(Length::Units(100))
	]
    )
```
Lets break that down a bit, first is a simple button, anyone who is familiar with iced will understand this part.
```rust
	button("I Grow and Shrink!")
	    .height(Length::Units(10))
	    .width(Length::Units(10))
```
Then we get to the new part, `.animation()` is how we know the widget can be animated. It is expected that all widgets with the `animation` method accept `iced::animation`. It is nothing more than a type that contains a `Vec` of `Keyframe`s, and does animation calculations and associated functions to simplify the process for each widget.

### Keyframes
What matters most to the end user are `Keyframe`s. They are a type per widget that says what can be animated on each widget. For example if a `button::Keyframe` has the method `.width()` then the widget's width can be animated. This lets each widget have it's own values, without the developer accidentally animating the wrong attribute, and without wasting data. The usage of keyframes is inspired by video editing software. In such software a keyframe can be placed at different durations through the video and attributes can be "attached" to the keyframe. The video editing software will then interpolate between each keyframe. For example if we have two keyframes 30 seconds appart, the first with a width of 10 units, the second being thirty seconds later with a width of 20, when we are at 15 seconds, we are halfway between the values, so the width will be 15 units. This assumes a linear interpolation, but we will leave that calculation for later. 
The same idea applies with this API, we are building a timeline of the animation. The animation could be just a simple interpolation between to values, or 100, there is no logical limit. Not every keyframe needs to have each value. For example,
```rust
	animation![
		button::Keyframe::new(Duration::from_secs(3))
		    .height(Length::Units(10))
		    .width(Length::Units(100))
		button::Keyframe::new(Duration::from_secs(6))
		    .width(Length::Units(10))
		button::Keyframe::new(Duration::from_secs(9))
		    .height(Length::Units(100))
		    .width(Length::Units(100))
	]
```
will animate the height constantly from 10 units to 100 for 6 seconds, while the width will animate from 100 units to 10 in 3 seconds, then back to 100 in another 3 seconds.

Where does the animation start from if the first `Keyframe` isn't `Duration::ZERO`? The animation will automatically prepend the animation with the current state of the widget. In out first example:
```rust
button("I Grow and Shrink!")
    .height(Length::Units(10))
    .width(Length::Units(10))
```
is the same thing as adding
```rust
	button::Keyframe::new(Duration::ZERO)
	    .height(Length::Units(10))
	    .width(Length::Units(10))
```
to the beginning of the `animation`. Why do it this way? I believe that this guides the developer in the direction of thinking of the widget as having a static layout, and the animation is added on top. If the animation is removed, the widget will return to its clearly defined "static" position. 

Note: If two `Keyframe`s have the same time, the interpolation will interpolate to the first one in the `Animation`, then the animation will suddenly jump to the second `Keyframe` with the same time.

## Implementation strategy

This implementation is broken into two parts widget repackaging, and widget redraw requests, eventhough they are both tied closely together.

### Widget Repackaging

Widget repackaging is a step where the widget's state from the previous render and the current widget are available at the same time. This allows for data to be moved/copied into the widget, for example a widget's cache, or where calculations can be done with data from the previous render, like interpolating an animation. This part is rather simple, Iced already `diff`s the current render with the previous render's state (cache). The change here is replacing the `diff` when building the new `UserInterface` with a mutable version. This does two things (1) gives a mutable reference to the widget (2) returns an accumulator, but (2) will be covered below in "Widget Redraws". In `diff_mut` the only difference is we call a repackage function on the widget before we continue to diff it's children. The trait function looks like so
```rust

      fn repackage(
        &mut self,
        state: &mut tree::State,
      ) {}
```
This function is deliberetly left vauge because each widget may have it's own process of cache that may be used. Though the general idea is the same for all, we are taking all data iced already holds about the widget and lets us repackage it into a new widget.

### Widget redraw requests

Widget redraws are a bit more complicated. Currently iced has two different types of widget state, `Some` and `None`. Exactly like an `Option`. Though we can expand on this enum with a few more:
```rust
pub enum State {
    /// No meaningful internal state.
    ///
    /// Retuning a state of this type will *not* cause `repackage` to be called on the widget the
    /// following render
    None,

    /// Some meaningful internal state.
    ///
    /// Retuning a state of this type will *not* cause `repackage` to be called on the widget the
    /// following render
    Some(Box<dyn Any>),
    
    /// A `Some` state, but the state will be reused the
    /// next render to to allow for values to be interpolated between redraws
    /// `AnimationFrame` means the widget is requesting a redraw as soon as reasonably
    /// possible (most likely at the time of the next monitor refresh).
    ///
    /// Retuning a state of this type will cause `repackage` to be called on the widget the
    /// following render
    AnimationFrame(AnimationState, Box<dyn Any>),
    
    /// A `Some` state, but the state will be reused the
    /// next render to to allow for values to be interpolated between redraws
    /// `Timeout` means the widget is requesting a redraw in the given `Duration`
    /// from widget interpolation.
    ///
    /// Retuning a state of this type will cause `repackage` to be called on the widget the
    /// following render
    Timeout(AnimationState, Instant, Box<dyn Any>),
    
    /// A `Some` state, but the state will be reused the
    /// next render to to allow for values to be cached for optimizations. This will not cause
    /// iced to renderer, it just makes the data available the next render, that must be triggered
    /// by some other event.
    ///
    /// Retuning a state of this type will will cause `repackage` to be called on the widget the
    /// following render
    Anytime(Box<dyn Any>)
}
```
Now the widget state can indicate if the widget has state, if it needs repackaging, and when it needs repackaging. We also do this in a way that enforces the data for an animation is available, but not mixed with the regular widget state.

#### Efficient and functional animation calculations
The `AnimationState` type in `State::AnimationFrame` and `State::Timeout` are as follows:
```rust
#[derive(Clone, Copy, Debug)]
pub struct AnimationState {
    /// The start time of the animation
    pub start: Instant,
    /// The hash of the animation. Used to check if the animation
    /// has changed since the previous animation started.
    pub hash: u64
}
```
A few points with this type (1) all data is Copy, so no heap allocations required for animations, (2) the animation itself isn't stored in the State, each render gets a clean `Animation` from the widget.
This works in a simple way, if the returned animation from `view()` is the same hash as the previous one we know it is the same animation, so we want to continue the animation from the previous renders. If the hash changes, we know that the developer has returned a new animation, thus we need to start from the beginning. All start is is the `Instant::now()`.
This is something that could be done in the current Iced API, but is inconvenient, and prone to failure (More detail in alternatives).
Using the difference from when the animation started to the time of render we can interpolate the animation value, all with pure data.

The widget can then let the iced runtime know when it would like to be rendered by by wrapping its data in the `State` type that is associated with when it wants to be redrawn. If it is a smooth animation like a resize, we want to redraw at the user's monitor's refresh rate (`State::AnimationFrame`), or it it is an animation that happens at a time like blinking cursor, or a gif at a lower refresh rate than the user's monitor we can request at an arbitrary time later (`State::Timeout`).

Iced's diff-ing process is recursive, thus we can just add an accumulator to the diffing process that returns the shortest time requested. If we have multiple widgets that want to redraw at the framerate that is fine, iced redraws everything all at once, meaning we catch all of them. If the redraw is earlier than a widgets requested `Timeout` that too is fine, it will just request a redraw at a later time again, until it is the widget (or tied with another widget) for needing to be the next one requiring a redraw.

#### How does Iced redraw?
Because we are repackaging widgets at the same time the `User Interface is created`, we can just store the accumulator there. For every Iced shell, like winit, a `UserInterface` is created each time an event is received. We can then just check the requested redraw time there, and pass it to the shell. For `iced_winit` it is simply an `Rc` that both `run_instance` can modify, and `run` can read, and then parse into the correct `ControlFlow` for winit.

#### Technical explanation of `Keyframe`s
A `Keyframe` is a trait that each widget will have to implement on it's own struct. The unique requirement is that all data needs to be converted into an array of usize (f32/64?) called modifiers. An example being
```rust
/// An animatable keyframe for a Row.
///
/// For iced internal devs:
/// modifiers:
/// [0] = width,
/// [1] = height,
/// [2] = spacing,
/// [3] = padding - top,
/// [4] = padding - right,
/// [5] = padding - bottom,
/// [6] = padding - left,
#[derive(Debug, Clone, Hash)]
pub struct Keyframe {
```
where a width might be attached to a `Keyframe` using a function like:
```rust
pub fn width(mut self, width: Length) -> Self {
self.modifiers[0] = Some((EASE, width.as_u16().unwrap() as isize));
self
}

```
I am not certain if there is a better way to handle this, but this allows for the complexities of the `Keyframe`'s timeline API, while allowing an automatic implementation of all interpolations with an unknown number of widget modifiers.

## Drawbacks

Some parts, particularly the Widget Repackaging might not be Elm-y enough. This may be true, but Elm lang itself has side effects with Gifs being able to redraw themselves, and this API itself could be created in Elm lang anyway.


## Rationale and alternatives

- Maintaining a subscription just for a blinking cursor is not a great UX. TEA is great for helping reduce simple easy to overlook mistakes. It would be easy for an Iced developer to forget to cancel the subscription when a text input loses focus, leading to unnecessary renders, that is a difficult to catch bug too.

- There are other iced animation crates that exist.


## Unresolved questions

- Should the animatable row/column/space widgets be different from the standard ones. If so could the `row!`/`col!` macros be expanded to return the correct version if an animation call is present or not? If so, what widgets should have this behavior.
- Iced theme animations, this would probably require getting a hold of the stylesheet in the widget repackaging step as well.
- should `Animation` be a trait rather than a struct? Would this be more efficient and just as easy for widgets to implement?


## [Optional] Future possibilities

- advanced easing
- physics animation
