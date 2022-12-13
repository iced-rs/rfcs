# Custom Shaders

## Summary

Add custom shader support to Iced under the `wgpu` feature.


## Motivation

Iced currently supports custom render pipelines, which allow users to define their own vertex and fragment shaders, but require much boilerplate code to do so. This RFC proposes a simpler API to define custom shaders for the specific case of rendering on a quad. The goal is to make it easier to render unique-looking widgets and decorations.


## Guide-level explanation

Custom shader quads are instanciated much like custom quads, but with two differences. The differences are that 

    1) they require a custom WGSL shader to be defined, and
    2) some custom attributes have to be bound to the shader.

A custom shader quad instance is created with the `make_custom_shader_quad(..)` method for the `Renderer` trait:

```rs
    fn make_custom_shader_quad(
        &mut self,
        custom_shader_quad: CustomShaderQuad,
        background: impl Into<Background>,
    );
```
where the `CustomShaderQuad` struct is defined as follows:

```rs
pub struct CustomShaderQuad {
    pub bounds: Rectangle,
    pub mouse_position: Point,
    pub mouse_click: Vector,
    pub time: f32,
    pub frame_number: u32,
    pub shader_code: String,
}
```

The `mouse_position`, `mouse_click`, `time`, `frame_number` fields are bound to the shader attributes behind the scenes, which means that their values are accessible in the shader code. The render can thus potentially react to mouse hovers, mouse clicks, key presses, etc. The names for `mouse_click`, `time`, `frame_number` are hard coded, but one could send any information to the shader through those fields provided that the types match (e.g. send an arbitrary flag through the `time` variable by setting it to either 0.0 or 1.0). The `shader_code` field expects the WGSL shader code that will be used to render the quad. 


To showcase custom shaders, we provide an example called `custom_shader_quad`, where the union of a star shape with a moon shape is rendered. In this example, the shadered quad interacts with mouse clicks and their timing, which means that we need
to pass this information to the shader. The shader code is imported from the `src` directory using 

```rs 
const SHADER: &str = include_str!("custom_shader.wgsl");
``` 

We define a struct called `StarMoon` that implements the `Widget` trait, and contains the following fields:

```rs
    pub struct StarMoon {
        size: f32,
        mouse_click: Vector,
        duration_since_start: Duration,
    }
```

In similar fashion to custom quads, the `draw(..)` method for the `Widget` implementation of `StarMoon` is written as follows:
```rs
        fn draw(
            &self,
            _state: &widget::Tree,
            renderer: &mut Renderer,
            _theme: &Renderer::Theme,
            _style: &renderer::Style,
            layout: Layout<'_>,
            cursor_position: Point,
            _viewport: &Rectangle,
        ) {
            renderer.make_custom_shader_quad(
                renderer::CustomShaderQuad {
                    bounds: layout.bounds(),
                    
                    mouse_position: cursor_position,
                    mouse_click: self.mouse_click,
                    time: self.duration_since_start.as_secs_f32(),
                    frame_number: 1,

                    shader_code: SHADER.to_string(),
                },
                Color::from_rgb(1.0, 0.0, 0.0),
            );
        }
```

On the shader side, the information is received in the form of attributes:

```rs
struct VertexInput {
    @location(0) v_pos: vec2<f32>,
    @location(1) pos: vec2<f32>,
    @location(2) size: vec2<f32>,
    @location(3) bg_color: vec4<f32>,

    @location(4) mouse_position: vec2<f32>,
    @location(5) mouse_click: vec2<f32>,
    @location(6) time: f32,
    @location(7) frame: u32,
}
```

The mouse clicks are encoded using a two-dimensional vector: the first component corresponds to a left mouse button press, and the second component to a right mouse button press. In this particular example, their values are zero at rest and one when pressed. In the shader code, the color of the shapes react to a right mouse button press:
    
```rs
    // upon left mouse button press, change border color
    if input.mouse_click.y > 0.5 {
        shape_color = vec4<f32>(0.84, 0.05, 0.92, 1.); // purple
    }

```



## Implementation strategy

The implementation of custom shader quads should follow that of custom quads as closely as possible. In summary, we modify the following modules: `graphics::renderer`, `wgpu::backend`, `graphics::primitive`, `graphics::layer` and `native::renderer`. And we add two new modules: `graphics::layer::custom_shader_quad` and `wgpu::custom_shader_quad`.

<!-- we add a new `Primitive` called `CustomShaderQuad` to the `graphics` module. 

We also add new `CustomShaderQuad` structs in  the `graphics::layer` module and in the `native::renderer` module. We add a new method called `make_custom_shader_quad(..)` inside the `Renderer` struct for instanciating the custom shader quad, and add a new struct to the `renderer` module. 

We add a new files to the the `graphics::layer` directory and the `wgpu` directory, both of which are named `custom_shader_quad.rs`. The new `graphics::layer` file contains the two structs `CustomShaderQuadWithCode` and `CustomShaderQuad`, the second of which can be directly converted to bytes. -->



We add a new  `graphics::Primitive`:
```rs
    CustomShaderQuad {
        bounds: Rectangle,
        background: Background, 
        mouse_position: Vector<f32>,
        mouse_click: Vector<f32>,
        time: f32,
        frame: u32,
        shader_code: String,
    },
``` 
followed by a new struct in `native::renderer`

```rs
#[derive(Debug, Clone, PartialEq)]
pub struct CustomShaderQuad {
    pub bounds: Rectangle,
    pub mouse_position: Point,
    pub mouse_click: Vector,
    pub time: f32,
    pub frame_number: u32,
    pub shader_code: String,
}
```


We create a new file called `custom_shader_quad.rs` in the `graphics::layer` directory, and add two new struct called  `CustomShaderQuadWithCode` and `CustomShaderQuad`:
```rs
pub struct CustomShaderQuadWithCode {
    pub position: [f32; 2],
    pub size: [f32; 2],
    pub color: [f32; 4],
    pub mouse_position: [f32; 2],
    pub mouse_click: [f32; 2],
    pub time: f32,
    pub frame: u32,
    pub shader_code: String,
}
```
```rs
pub struct CustomShaderQuad {
    pub position: [f32; 2],
    pub size: [f32; 2],
    pub color: [f32; 4],
    pub mouse_position: [f32; 2],
    pub mouse_click: [f32; 2],
    pub time: f32,
    pub frame: u32,
}
```
The first struct (`CustomShaderQuadWithCode`) contains all the information about the shader including its code. The second struct can directly be converted to bytes for sending to the GPU, since the `shader_code` (`String`) field is not present.



In `graphics::layer`, we add a new method for the `Layer` struct:
```rs
            Primitive::CustomShaderQuad {
                bounds,
                background,
                mouse_position,
                mouse_click,
                time,
                frame,
                shader_code,
            } => {
                let layer = &mut layers[current_layer];

                layer.custom_shader_quads.push(CustomShaderQuadWithCode {
                    position: [
                        bounds.x + translation.x,
                        bounds.y + translation.y,
                    ],
                    size: [bounds.width, bounds.height],
                    color: match background {
                        Background::Color(color) => color.into_linear(),
                    },

                    mouse_position: [mouse_position.x, mouse_position.y],
                    mouse_click: [mouse_click.x, mouse_click.y],
                    time: *time,
                    frame: *frame,
                    shader_code: shader_code.clone(),
                });
            }
```

The `flush(..)` method in `wgpu::backend` needs to be modified to handle the `CustomShaderQuad` layer:

```rs
        if !layer.custom_shader_quads.is_empty() {
            let serializable_instances: Vec<layer::CustomShaderQuad> = layer
                .custom_shader_quads
                .iter()
                .map(|x| layer::CustomShaderQuad::from(x))
                .collect::<Vec<layer::CustomShaderQuad>>();

            self.custom_shader_quad_pipeline.draw(
                device,
                staging_belt,
                encoder,
                &layer.custom_shader_quads,
                &serializable_instances,
                transformation,
                scale_factor,
                bounds,
                target,
            );
        }
```

In `graphics::renderer`, we add a new method for the `Renderer` struct:
```rs
    fn make_custom_shader_quad(
        &mut self,
        quad: renderer::CustomShaderQuad,
        background: impl Into<Background>,
    ) {
        self.primitives.push(Primitive::CustomShaderQuad {
            bounds: quad.bounds,
            background: background.into(),
            mouse_position: Vector::new(
                quad.mouse_position.x,
                quad.mouse_position.y,
            ),
            mouse_click: Vector::new(quad.mouse_click.x, quad.mouse_click.y),
            time: quad.time,
            frame: quad.frame_number,
            shader_code: quad.shader_code,
        });
    }
```



This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


## Drawbacks

Since the interaction information (mouse press, key press, etc) is sent through attributes, this information is sent once for every vertex, which amounts to four times the amount of data that is actually needed. Ideally, this information would be sent through a uniform buffer that is unique to each shader quad, but this would require a more complex implementation and would most likely break the current implementation of other rendering primitives. In the grand scheme of things, this amount of extra data  is not very large at the moment, but it is something to keep in mind since it wouldn't be practical to generalize the above scheme to more complex vertex geometries.


## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?


## [Optional] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other GUI toolkits and what experience have their community had?
- Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other toolkits, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that iced sometimes intentionally diverges from common toolkit features.


## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?


## [Optional] Future possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the toolkit and project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
