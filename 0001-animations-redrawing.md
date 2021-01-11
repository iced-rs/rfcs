## Animations 
Animations can either modify what the widget draws, or also modify the widgets layout properties.
Our animation system needs to be able to handle both of these cases.

For both of these use-cases we need to allow the widget to express when it wants to be redrawn/layouted next.

### Denoting desire to be redrawn
Lets add a new structure called 

```rust
/// Denotes when the widget wants to be redrawn/reloayouted next
pub enum NextParams {
    /// The widget wants to be redrawn on next frame
    NextFrame,
    /// The widget wants to be redraw at most at the specified time, or earlier
    At(Instant),
    /// The widget doesn't care about redrawing
    DontCare
}
```
The ordering of `NextParams` is `NextParams::NextFrame < NextParams::At(0) < NextParams::At(1) < NextParams::DontCare`

We then change  `Widget::draw` to:
```rust
 fn draw(
    &self,
    renderer: &mut Renderer,
    defaults: &Renderer::Defaults,
    layout: Layout<'_>,
    cursor_position: Point,
    viewport: &Rectangle,
) -> (Renderer::Output, NextParams);
```


Then, when each widgets calls `draw` on its children, it returns `std::cmp::min` of the returned `NextParams` denoting 
earliest time, when the widget needs to be redrawn.

This is then propagated through the implementation of `UserInterface` to the `run_instance` method, which contains the event loop logic.

## Responding to requests to be redrawn.
All of these changes are in `application.rs` of a specific backend. The backend takes the returned `NextParams` into account, when the backend should redraw/relayout the view hierarchy.

#### For winit backend

We modify `run_instance` to take the `NextParams` returned from user interface into account. This information denotes the desire of the user interface to be redrawn. 

We change the `run_instance` from returning a Future, to a method returning a `Stream<Item=NextParams>`, this is then handled by the `run` method, which in turn calls `poll_next` on the stream returned from `run_instance` and changes the `ControlFlow` returned to the winit event loop.

```rust
*control_flow = match poll {
    task::Poll::Pending | task::Poll::Ready(Some(NextParams::Dontcare)) => ControlFlow::Wait,
    task::Poll::Ready(Some(NextParams::NextFrame)) => ControlFlow::Poll,
    task::Poll::Ready(Some(NextParams::At(instant))) => ControlFlow::WaitUntil(instant),
    task::Poll::Ready(None) => ControlFlow::Exit,
};
```

This should make it available for views to denote what kind of redrawing/relayouting behavior they expect.

### Draw animations
For TextInput, we want to blink the cursore every 500ms since the editText was selected. We can implement this by adding
a following field to TextInput state
```rust
selected_at : Instant
```
Then, in draw we get current time, and only draw the cursor, if the following expression is true:
```
((state.selected_at - Instant::now()).as_millis() / 500) % 2 == 1
```
Thus, drawing the cursor only every 500ms.

### Attribute animations
Second type of animations deals with animating attributes, which can affect both layout and draw tree.
For these, we need to run whole view-layout-draw loop.

We can support these animations with following widget:
```rust
pub struct Animate<T: Widget, F : FnOnce(&T, Duration) -> T> {
    // Stores start time, end time, and the general state of the animation
    state: &mut iced::widget::animate::State,
    // Stores the underlying widget we want to animate
    widget: T,
    // Stores the function used for animation
    animator: F
}
```
which has methods to animate individual properties of the widget. The `Animate` widget creates a new widget instance for each frame where it needs to animate. This allows for animation without introducing mutability into the draw/layout process.


## Conclusion
Steps we need to do:
1. Introduce `NextParams` into `Widget::layout` and/or `Widget::draw`.
2. Return `min(children_params...)` in widget implementations
3. Use this information to request complete relayout/redraw from the runtime.
4. Use runtime capablity for requesting event loop iterations at specific times.
5. Implement animations depending on widget state and current time in `Widget::layout, Widget::draw`

### Issues
This is just a slightly messy, inefficient outline of a system for supporting animations. This system would work even 
without persistent widget tree, allowing the widgets to request re-creation when needed.
