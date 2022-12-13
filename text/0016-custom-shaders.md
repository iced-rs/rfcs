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

![custom_shader_quad](/text/custom_shader_quad.gif)


To showcase custom shaders, we provide an example called `custom_shader_quad`, where the union of a star shape with a moon shape is rendered. See the example here: https://github.com/eliotbo/iced/tree/master/examples/custom_shader_quad. In this example, the shadered quad interacts with mouse clicks and their timing, which means that we need
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
This struct should be called by the end user to instanciate a custom shader quad. It is present inside the `draw(..)` method in the example above.

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

This `make_custom_shader_quad(..)` method must be called by the end user to instanciate a custom shader quad. It is also present inside the `draw(..)` method in the example above. In the case where the `mouse_position`, `mouse_click`, `time` and `frame` fields are not needed, the user must give dummy values to these fields anyway.

For all widgets that implement the `Overlay` trait, one could use a custom shader by calling `make_custom_shader_quad(..)` instead of `fill_quad(..)` on the `Renderer` inside the `draw(..)` method of the `Overlay` trait.


## Drawbacks

Since the interaction information (mouse press, key press, etc) is sent through attributes, this information is sent once for every vertex, which amounts to four times the quantity of data that is actually needed. Ideally, this information would be sent through a uniform buffer that is unique to each shader quad, but this would require a more complex implementation and would most likely break the current implementation of other rendering primitives. In the grand scheme of things, this quantity of extra data  is not very large, but it is something to keep in mind since it wouldn't be practical to generalize the above scheme to more complex vertex geometries.

Custom shaders would only be available with the `wgpu` feature. 

The byte size of the data going from the CPU to the GPU is fixed through attributes is fixed at 26 bytes right now. Users cannot send more data than this to their custom shader.

## Rationale and alternatives

Currently, the alternative to custom shader quads is to implement a custom WGPU pipeline with a lot of boilerplate, but with more flexibility. The custom shader quads allow users who are not familiar with GPU pipelines to use custom shaders rather easily.

The impact of *not* implementing this feature is that it is harder to personalize the look of an Ice application. For example, there is only one plotting library that is compatible with Iced at the moment, and it might not produce a good enough look for a particular class of commercial products like audio plugins. Shaders give users the ability to improve the look of their applications tremendously.




## Unresolved questions

The implementation is already up and running in my fork of the `iced` repository ( https://github.com/eliotbo/iced ), but there are a few questions that I would like to discuss.

If one were to use a custom shader quad as a transparent `Overlay` to replace the default look of a widget, how would one hide the default look of the widget? We could probably solve this by adding a `visibility` field in the `Appearance` of a widget, but this is a very significant change to the library.

How to use unique uniforms for each quad: see first paragraph of the Drawbacks section.


## Future possibilities

It would be better to find a more flexible way to send data to the GPU. Again, see first paragraph of the Drawbacks section.

A more thorough implementation could be done for the `glow` backend as well, which I happen to not know at all unfortunately.
