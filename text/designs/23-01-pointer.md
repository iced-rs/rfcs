# 🙌 #1: Custom Primitive Pointer Design

You can view a small, very, *very* rough and unrefined prototype of this design [here](https://github.com/bungoboingo/iced/tree/custom-shader/pipeline-marker/examples/custom_shader/src).

The design of this implementation focuses on providing a fn pointer which both initializes the state of a custom
pipeline, and returns a pointer to that state so the backend can cache & reuse. We would need to expose a type of
trait which has methods necessary for rendering, something along the lines of:

```rust
pub trait Renderable {
    fn prepare(
        &mut self,
        _device: &wgpu::Device,
        _queue: &wgpu::Queue,
        _encoder: &mut wgpu::CommandEncoder,
        _scale_factor: f32,
        _transformation: Transformation,
        _time: Duration, //used for shader animations, calculated every frame
    );

    fn render(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        _device: &wgpu::Device,
        _target: &wgpu::TextureView,
        _clear_color: Option<Color>,
        _scale_factor: f32,
        _target_size: Size<u32>,
        // final implementation may contain more information
    );
}
```
And have the ability to initialize a pipeline & retrieve a pointer to it's state:

```rust
pub init: fn(
    device: &wgpu::Device,
    format: wgpu::TextureFormat,
) -> Box<dyn Renderable>;
```

Which would need to be stored in a `Primitive`, e.g. `Primitive::Custom`, which could be used by a user like this:

```rust
    renderer.draw_primitive(Primitive::Custom {
        bounds,
        pipeline: CustomPipeline {
            id: self.id, // a pipeline identifier so we can look up the data pointer
            init: State::init, // an initialization fn pointer which returns a pointer to the pipeline data
        },
    })
```

This would be the entirety of the API exposed to a user of Iced. The rest of the implementation details would be
handled internally by the wgpu `Backend`.

For example, a typical implementation from a user in this "hands off" scenario might look something like this:

```rust
pub struct Pipeline {
    pipeline: wgpu::RenderPipeline,
    vertices: wgpu::Buffer,
    indices: wgpu::Buffer,
    // ...
}

impl Pipeline {
    // We must provide a way for the Renderer to initialize this pipeline since it needs to hold a 
    // pointer to this `State`
    fn init(
        device: &wgpu::Device,
        format: wgpu::TextureFormat,
        target_size: Size<u32>,
    ) -> Box<dyn Renderable + 'static> {
        let vertices = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some("cubes vertex buffer"),
            size: std::mem::size_of::<[Vertex3D; 8]>() as u64,
            usage: wgpu::BufferUsages::VERTEX | wgpu::BufferUsages::COPY_DST,
            mapped_at_creation: false,
        });

        //... rest of the wgpu pipeline initialization omitted for brevity!

        Box::new(Self {
            pipeline,
            vertices,
            indices,
        })
    }
}

/// Implement the "renderable" trait for this `State` struct
impl Renderable for Pipeline {
    fn prepare(
        &mut self,
        _device: &wgpu::Device,
        _queue: &wgpu::Queue,
        _encoder: &mut wgpu::CommandEncoder,
        _scale_factor: f32,
        _transformation: Transformation,
        _time: Duration,
    ) {
        // Allocate what data you want to render
        let mut cube = Cube::new();
        queue.write_buffer(&self.vertices, 0, bytemuck::bytes_of(&cube));
    }

    fn render(
        &self,
        encoder: &mut wgpu::CommandEncoder,
        _device: &wgpu::Device,
        _target: &wgpu::TextureView,
        _clear_color: Option<Color>,
        _scale_factor: f32,
        _target_size: Size<u32>,
    ) {
        let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            /// omitted for brevity
        });

        render_pass.set_pipeline(&self.pipeline);
        // issues command to the render_pass
    }
}

/// Include the custom "primitive" into a custom widget
struct Cubes {
    height: Length,
    width: Length,
    id: u64, // a pipeline identifier
}

impl<B: Backend, T, Message> Widget<Message, iced_graphics::Renderer<B, T>> for Cubes {
    // ...

    fn draw(
        &self,
        _state: &Tree,
        renderer: &mut iced_graphics::Renderer<B, T>,
        _theme: &T,
        _style: &Style,
        layout: Layout<'_>,
        _cursor_position: Point,
        _viewport: &Rectangle,
    ) {
        ///..
        /// now pass the pointer to the renderer along with a unique pipeline ID for caching & lookup
        renderer.draw_primitive(Primitive::Custom {
            bounds,
            pipeline: CustomPipeline {
                id: self.id,
                init: Pipeline::init,
            },
        })
    }
}
```

```rust
/// In your application code..
fn view(&self) -> Element<'_, Self::Message, Renderer<Self::Theme>> {
  Cubes::new() // Initiate your custom widget which draws a custom primitive
      .width(Length::Fill)
      .height(Length::Fill)
      .id(0) // set a pipeline ID so we can store a pointer to a specific "renderable" state
      .into()
}
```