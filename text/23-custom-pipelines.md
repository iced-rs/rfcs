# 🌈 Custom WGPU Shaders & Pipelines 

## 📜 Summary

This RFC aims to reach a consensus on the following:
1) Should we integrate custom WGPU shader & pipeline support into Iced?
2) If so, to what extent? And how?

This RFC is similar to [RFC #19](https://github.com/iced-rs/rfcs/pull/19), but with a broader scope (for example, the 
custom shader proposed in #19 could be built on top of this framework!). The idea is to allow users to use their 
own WGPU pipelines & custom primitives to render their own content seamlessly alongside Iced's existing supported primitives.

**Please note that this RFC is working off of the changes already present in the `advanced-text` branch!**

## 🦾 Motivation

Currently, the only way to use a custom shader with Iced is to do something like the WGPU 
`integration` example, where you can render your own scene using Iced *independently* of the existing widget tree, 
or create a custom `Renderer` and do everything yourself. You must either render your content before or after Iced 
renders its user interface. This works in niche situations, for example a custom 3D scene with a 2d Iced overlay 
rendered on top, but if you wish to adjust the shader used to fill, for example, a container that exists deep 
within a widget tree, this is currently impossible without a fork or recreating the effect with potentially 
very expensive CPU calculations on a `Canvas`.

Adding custom WGPU shader & pipeline support to Iced will add extensibility, customizability, and modularity to the
existing rendering pipeline. I imagine a near future where we have repositories similar to `iced_aw`, where library
authors can provide their own pipeline integrations into Iced for all kinds of shaders, primitives, and even
post-processing effects! 🤯

Most GUI libraries out there allow some way of embedding the underlying graphics API calls alongside existing 
widgets for further customizability, an example being querying a raw `WebGL` context from a HTML5 canvas and 
doing what you will with it. I believe Iced would benefit immensely from this added functionality.

The result of this RFC should be that there is a clear path for allowing "embeddable" WGPU pipelines in some fashion 
to Iced, where library authors can create their own primitives & rendering pipelines which integrate seamlessly into 
Iced's existing renderer & backend.

## 🤠 Guide-level explanation

### 📚 Concepts

*Foreword:* Coming up with a strategy for adding custom pipeline support to Iced is not really about what, nor how, but 
*when*.
At what 
point in the frame presentation does the custom primitive get inserted into the primitive queue? Does it even get 
inserted into the primitive queue at all? If it's not inserted, how do we know when to render it? If it is inserted, 
what does a custom primitive actually mean? What *is* it? How do we reference the correct pipeline from the custom 
primitive? If the custom primitive has its own primitive type, do we use codegen/generics and just take the generic 
propagation L? Do we use reflection? Do we use trait objects & dynamic dispatching? Do we completely restructure 
Iced's rendering pipeline to allow for multiple backends of the same type? If so, how do we allow for any arbitrary 
amount of additive backends? Do we start getting down & dirty with macros to shave off a few additional pointer 
hops? Could we do this with a minimal unsafe abstraction layer? Should we even do this at all? Why am I doing this? 
*Why do I even exist?*

![](silvia.jpeg)

As you can see, this is a somewhat complex topic with a lot of tradeoffs between implementation strategies. Of 
course, I would love if there was a better idea floating around out there that I haven't thought of! That being said,
here are a few different designs & their concepts that would need to be introduced to Iced. I've broken them into 
their own markdown files for an easier time reading!

1) [Custom Primitive Pointer Design](designs/23-01-pointer.md)
2) [Custom Shader Widget Design](designs/23-02-widget.md)
3) [Multi-Backend Design](designs/23-03-multi-backend.md)

All of these designs must be flagged under `wgpu`, unless we wanted to do some kind of fallback for tiny-skia which 
I don't think is viable. What would we fall back to for the software renderer if a user tries to render a 3D object, 
which tiny-skia does not support? Blue screen? :P

Overall, I'm the most happy with design #3 and think that it offers the most flexibility for advanced users to 
truly render anything they want.


## 🎯 Implementation strategy

### 🙌 #1: Custom Primitive Pointer Design

Behind the scenes, this would require very little changes to Iced! 

A `Primitive::Custom` variant would need to be added to the existing `iced_graphics::Primitive` enum, in order to 
have a way to pass a pipeline pointer to the `Renderer` and indicate its proper order in the primitive stack.

We would also need to add a new field to an `iced_wgpu::Layer`, something along the lines of:

```rust
pub struct Layer<'a> {
  //...
  custom: Vec<PipelineId>,
}
```

To indicate which pipelines are grouped within this layer. Or perhaps we could require that all custom pipelines are 
on separate layers, though that has performance implications.

We would also need a way to cache & perform lookups for trait objects which implement `Renderable`; in my prototype 
I've simply used a `HashMap<PipelineId, Box<dyn Renderable>` inside of the `iced_wgpu::Backend`.

Then, when rendering during frame presentation, we simply perform a lookup for the `PipelineId`s contains within the 
`Layer`, and perform their `prepare()` and `render()` methods. Done!

✅ **Pros of this design:**

- Simple to integrate into existing Iced infrastructure without major refactors.
- Performance is acceptable

❌ **Cons of this design:**

- Not flexible
  - Even with preparing this very simple example I found myself needing to adjust the `Renderable` trait to give me 
    more and more data unique to that pipeline that I needed for that specific render pass to render a cube.
- Feels kinda hacky
  - `Primitive::Custom` feels as though it doesn't really belong in the existing `iced_graphics::Primitive` enum, 
    but that's subjective!
- `prepare()` and `render()` calls must be dynamically dispatched every frame per-pipeline & thus cannot be inlined.

Overall I'm pretty unhappy with this implementation strategy, and feel as though it's too narrow for creating a truly 
flexible & modular system of adding custom shaders & pipelines to Iced.

### 🎨 #2: Custom Shader Widget

The internals for this custom shader widget are very similar to the previous strategy; the main difference is that 
internally, *we* would create the custom primitive which holds the pointer to the pipeline data, not the user. The 
other difference is that the `Program` trait is merged with the `Renderable` trait, and that we create the widget 
implementation for the user, no custom widget required.

Other concepts must be added, like the concept of `Time` in a render pass. In my prototype, I've implemented it at 
the `wgpu::Backend` level, but in its final form we would need to shift it up to the `Compositor` level, I believe. 
It's exposed to the user as a simple `Duration`, which is calculated from the difference between when the 
`iced_wgpu::Backend` is initialized up until that frame.

There may be other information needed in the `Program` trait which is discovered as the implementation evolves.

✅ **Pros:**

- Users who are already familiar with `Canvas` might find this type of widget familiar & intuitive to use.
- More in line with Iced's style

And, like the previous strategy:
- Simple to integrate into existing Iced infrastructure without major refactors.
- Performance is acceptable

❌ **Cons:**
- Same cons as the previous strategy; very little flexibility, users must shoehorn their pipeline code to fit into 
  this very specific trait `Program` provided by Iced.

### 🔠 Multiple Backend Support for Compositors

Internally, this design is the most complex and requires the most changes to Iced, but I don't think it's so wildly 
complex that it would be hard to maintain! This design would require a few new concepts added to the wgpu `Compositor`:

💠 **Primitive Queue**

There must be a backend-aware queue which keeps track of the actual ordering of how primitives should be
rendered across all backends. I believe this could be implemented fairly easily either by having each `Backend` keep
track of its own queue and having some data structure delegate at the appropriate moment with some form of marker
indicating that we need to start rendering on another `Backend`. Some kind of order-tracking data structure is
essential for ensuring proper rendering order when there are multiple backends.

Widgets would request that their custom primitives be added to this queue when calling `renderer.draw_primitive()`.

👨‍💼 **Backend "Manager"**

This would essentially be responsible for initializing all the backends (lazily, perhaps!) & delegating the proper
primitives to the multiple `Backend`s for rendering. This would be initialized with the `Compositor` on application
start.

👨‍💻 **Declarative backends!() macro**

This would be initialized in `Application::run()` as a parameter, or could be exposed somewhere else potentially 
(perhaps as an associated type of `Application`?). I haven't super thoroughly thought it through, but my initial 
idea is to have it return a `backend::Manager` from its `TokenStream` which would be moved into the `Compositor`.

✅ **Pros:**
- Flexible, users can do whatever they want with their own custom `Backend`.
- Modular & additive; users can create a custom `Backend` library with their own primitives that it supports that 
  can be initialized with the  `Compositor`.
- For users wanting to use a custom primitive from another library, or one they made, they would use it exactly how  
  you use currently supported `Primitive`s in Iced, which would feel intuitive.

❌ **Cons:**
- Would involve a hefty amount of codegen to do performantly
- This would be quite a heavy refactor for the `iced_wgpu::Compositor`!
- This design would preclude custom primitives being clipped together with other backend's primitives in 
  its own `Layer` for transformations, scaling, etc. which might be undesirable. There might be a way to implement 
  this within the `backend::Manager`, however!

### 🤔 Other Ideas

I've had several ~~thousand~~ other ideas that I thought I'd give a brief mention. I've dismissed most of these as 
not being viable, but perhaps someone can think of a better way to implement them.

1) Primitive Concatenation with `#[primitive]` attribute macro

This idea would involve heavily ~~ab~~using proc macros to essentially "embed" Iced's primitives into a user's custom 
primitive type, allowing them to set the primitive type of a `Renderer` at compile time, and include Iced's already 
supported primitives, without needing dynamic dispatch. Library authors of custom primitives & pipelines could then 
provide their own macro, which would be embedded into the `#[primitive]` attribute macro to keep concatenating 
enums until you had a final `Primitive` type. This however feels like more of a hack than a real solution, at least 
to me!

2) Fck it, build script

This idea would involve a hefty amount of codegen to generate both the final `Primitive` type similar to above, and 
also the whole `Backend` with any custom pipelines embedded in it. This way a user has access to the exact same API 
as currently offered in Iced when working with custom primitives. Also dismissed due to hackiness, but some sort of 
build script might still be needed in the end (hopefully not to this extent!).

## 😶‍🌫️ Drawbacks

After prototyping out ideas for the last 2 weeks, I have come to the conclusion that whatever strategy we 
decide, unless we do a hefty bit of codegen, will ultimately be less performant than just forking Iced & adding a 
new pipeline/primitive directly to the existing WGPU backend. I'm of the opinion it's not a bad thing to fork a 
library if you want to add something that others might not want, so perhaps that is the real solution. It's 
certainly the simplest!

You *can* also use custom shaders right now with Iced, albeit in a limited way, so perhaps further integration is 
not needed. Perhaps we should look at a more composable way of creating a `Renderer` that can leverage already 
implemented support from Iced's `Renderer`s instead, or something along those lines.


## 🧐 Rationale and alternatives

I believe I've already addressed most of these pros & cons in the `Implementation` section above! I will say that the 
implications of *not* doing this is that Iced would continue to be one of the few GUI libraries that does not offer 
direct access to its rendering backend (e.g. the `wgpu::Device` and `wgpu::CommandEncoder`) or just the ability in 
general to embed graphics-api-specific content in its existing widget tree. I believe that not allowing users to 
leverage the GPU for more complex scenes alongside the ease of using Iced's built-in widgets and rendering pipelines 
would be tragic! 😿


## 🧑‍🎨 Prior art

The most prevalent example of this feature that I can think of is being able to get a direct `WebGL` context from a 
HTML5 `Canvas`, which allows users to render whatever they want (except things that use storage buffers! :P) on the 
web. This has allowed all kinds of content for the web that we didn't have before, like complex 3D scenes, whole 
games (RIP Flash), etc. that I think have made the web richer. Of course there is a performance implication of 
allowing such raw access on a platform which some might say is unoptimized for such tasks, but that is not really a 
consideration with Iced.

A lot of other GUI frameworks allow use of custom shaders via OOP & inheritance (for example, by simply extending a 
`Primitive` class or interface which defines how to draw itself). Obviously Iced's implementation will need to 
differ since it is a functional library (although some form of trait object might be needed). Any game engine out there 
allows custom shaders & entities, all submitted to one (or multiple) render queues.

Android allows for "shaders" using their own shader language, AGSL, which functions similar to Iced's `Canvas`, but 
also with GLSL and hooking directly into the GL context. They also use OOP & inheritance with a `GlSurfaceView` which 
extends `View` (which is the base abstract class for all Android widget tree elements). This allows users to 
interact with the (E)GL surface which provides a bounds that they can execute OpenGL commands within.

The common thread amongst all of these implementations is that advanced users can access a raw handle to the 
graphics backend and draw to a specific region that is integrated into the rest of the DOM or ECS or widget tree or 
w/e to enable advanced drawing capabilities leveraging the GPU. I think the benefits of this type of flexible 
integration are readily apparent!

## 😵‍💫 Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

I would expect that this RFC resolves the overall direction of integrating flexible custom shader & pipeline support 
into Iced. I'd like to discuss & settle on a game plan before committing real time to the final implementation.

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

The exact implementation of the intermediary data structures (if needed!) for managing rendering order & backends, 
small things that pop up during implementation that weren't considered. Once the general design is agreed upon I 
think the implementation details can be iterated on in PRs or through discussion in this RFC/Discord.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

I think any actual implementations of custom shaders would be built on-top of this RFC, possibly in a separate crate 
within the Iced organization.

## 🤖 Future possibilities

I think getting this design implemented correctly would open a huge number of doors for the Iced ecosystem. 
Being able to write your own shaders and create a custom widget which any user can plug into their own application 
and use seamlessly as part of Iced's widget tree is the ultimate form of customization.

### If you made it to the end, congratulations! 🥳

Now, let's discuss!

