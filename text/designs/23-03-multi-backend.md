# 🔠 Multiple Backend Support for Compositors

I have no prototype to speak of this with strategy; it will involve a good amount of restructuring, possibly some
codegen for performance reasons, and some intermediate data structures added to the compositor. That being said, I
believe this is more along the lines of a "correct" solution for integrating custom shaders & pipelines into Iced as
it allows the most flexibility & feels the least hacky.

This strategy involves adding support for multiple `Backend`s per `Compositor`. See the diagram below for a rough
outline of how it would work:

![](diagram.png)

A user or library author would be responsible for creating their own `Backend` data structure that handles 
all primitives of a certain type.

```rust
struct CustomBackend {
    // pipelines that the user has created!
    pipeline_3d: Pipeline3D,
    //...
}

impl Backend for CustomBackend {
    type Primitive = CustomPrimitive;
    type Layer = CustomLayer;
}

pub enum CustomPrimitive {
    Sphere(Sphere),
    Cube(Cube),
    //...
}

struct CustomLayer {
    pub spheres: Vec<Sphere>,
    pub cubes: Vec<Cube>,
    //...
}
```
This `CustomBackend` would need to implement a certain trait type, here named `Backend`, which could be defined 
something like this:

```rust
pub trait Backend {
    type Primitive;
    type Layer;
    
    fn present(
        &mut self,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        encoder: &mut wgpu::CommandEncoder,
        clear_color: Option<Color>,
        format: wgpu::TextureFormat,
        frame: &wgpu::TextureView,
        primitives: &[Self::Primitive],
        viewport: &Viewport,
    );

    fn prepare(
        &mut self,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        encoder: &mut wgpu::CommandEncoder,
        scale_factor: f32,
        transformation: Transformation,
        layers: &[Self::Layer<'_>],
    );

    fn render(
        &mut self,
        device: &wgpu::Device,
        encoder: &mut wgpu::CommandEncoder,
        target: &wgpu::TextureView,
        clear_color: Option<Color>,
        scale_factor: f32,
        target_size: Size<u32>,
        layers: &[Self::Layer<'_>],
    );
    
    // other methods might be needed!
}
```

Users would include this send this backend to the Compositor in some way, either at runtime (box'd, dyn'd) with a 
`Command`, or I was thinking of a more performant solution involving codegen, either a build script or with a 
declarative macro, something like:

```rust
Application::run(
    iced::Settings::default(),
    //.. other backends provided by library authors could be added in this declarative macro!
    backends!(Iced, CustomBackend, OtherBacked,),
)
```

From a user's perspective, that's it! Just including the backend would map it internally to a custom primitive type, 
and then they would be able to draw a custom primitive same as any other Iced primitive.

```rust
//...
renderer.draw_primitive(
    CustomPrimitive::Sphere(Sphere {
        u_sections: 16,
        v_sections: 16,
        radius: 4.0,
    })
)
```

