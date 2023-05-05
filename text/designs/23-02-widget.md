# 🎨 #2: Custom Shader Widget

You can view a rough prototype of this strategy here: [here](https://github.com/bungoboingo/iced/tree/custom-shader/widget/examples/custom_shader/src)

Similar to how we currently have `Canvas` in Iced, this design would involve exposing a new built-in widget which is
dependent on `wgpu` that has its own `Program` where a user can define how to render their own data.

A custom shader widget might have a `Program` trait similar to a `canvas::Program`, but with methods from the
`Renderable` trait from strategy #1.

```rust
pub trait Program {
    fn update(
        &mut self,
        _event: Event,
        _bounds: Rectangle,
        _cursor: Cursor,
        _device: &wgpu::Device,
        _queue: &wgpu::Queue,
        _encoder: &mut wgpu::CommandEncoder,
        _scale_factor: f32,
        _transformation: Transformation,
        _time: Duration,
    ) -> (event::Status, Option<Message>);

    fn render(
        &self,
        _encoder: &mut wgpu::CommandEncoder,
        _device: &wgpu::Device,
        _target: &wgpu::TextureView,
        _clear_color: Option<Color>,
        _scale_factor: f32,
        _target_size: Size<u32>,
    ) -> RenderStatus;

    fn mouse_interaction(
        &self,
        _state: &Self::State,
        _bounds: Rectangle,
        _cursor: Cursor,
    ) -> mouse::Interaction {
        mouse::Interaction::default()
    }
    
    //possibly some more needed methods 
}
```

Similar to `Canvas`, but without a `draw()` method; instead there is a `render()` method which returns a new enum, 
`RenderStatus` (name TBD). Might need to change the name `Program`, as `Shader::Program` is an overloaded term!

Users would return either `RenderStatus::Done` or `RenderStatus::Redraw` to indicate their render operation either 
can wait to be redrawn until the next application update, or must be redrawn immediately. This is the case when a 
shader has an animation.

New to Iced & a parameter of the `update()` method is the concept of `time`. This is simply a duration of time that 
has passed since the start of the application. This can be used to animate shaders (see my prototype above for an 
example!).

We will probably also end up using an associated `State` type similar to `Canvas` for handling internal state mutation.

A user will create a new custom shader widget using the `Shader` widget implementation in `iced_graphics`.

```rust
pub struct Shader {
    width: Length,
    height: Length,
    init: fn(
        device: &wgpu::Device,
        format: wgpu::TextureFormat,
        target_size: Size<u32>,
    ) -> Box<dyn Program + 'static>,
    id: u64, //unique pipeline ID
    //properties subject to change!
}
```

This provides the `Program` pointer, similar to the "custom primitive pointer design" I've listed in the previous file. 
Users will simply implement the `Program` trait for their own data structure & pass the initializer to the `Shader` 
widget.

```rust
    fn view(&self) -> Element<'_, Self::Message, Renderer<Self::Theme>> {
        Shader::new(Pipeline::init, 0)
            .width(Length::Fill)
            .height(Length::Fill)
            .into()
    }
```

Where `Pipeline::init` creates the pipeline code for wgpu. This is about all that is exposed to a user of the 
library! The rest is internal implementation details.
