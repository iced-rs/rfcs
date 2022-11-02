# SVG Theming

## Summary

The ability to specify the fill color of a SVG icon.

## Motivation

Symbolic icons are SVGs which define a single color with varying alpha transparency to every path. GUI themes can utilize these to adjust their colors against the background, either to support dark and light theme variants, or to apply extra emphasis to make the icon stand out against the rest, or even de-emphasize options that are disabled by darkening or lightening the color closer to the background color.

For example, the UX for libcosmic will apply a darker blue color for icons in the headerbar while using the light variation of the theme, whereas the dark variant of the theme will use a lighter blue color. Meanwhile, the rest of the symbolic icons will be assigned to a bright white color on the dark theme and a dark black on the light theme. While disabled options will have a brighter black on the light theme and a darker white on the dark theme.

## Guide-level explanation

This will be most useful for the processing of symbolic icons. Therefore, the first step is to load a symbolic icon from an icon theme. On Linux, icons from a designated icon theme can be located using the `freedesktop-icons` crate.

```rs
let handle: svg::Handle = freedesktop_icons::lookup("window-close-symbolic")
    .with_size(24)
    .with_theme("Pop")
    .with_cache()
    .force_svg()
    .find()
    .map_or_else(
    	|| svg::Handle::from_memory(Vec::new()),
    	svg::Handle::from_path
    );
```

This SVG may then have its fill color altered by defining a custom appearance:

```rs
let svg = iced::widget::svg::Svg::new(handle)
    .width(Length::Units(24))
    .height(Length::Units(24))
    .style(iced_style::theme::Svg::Custom(|theme| {
        match theme {
            Theme::Light => Appearance {
                fill: Some(Color::from([0, 0.28627452, 0.42745098]),
            },
            Theme::Dark => Appearance {
                fill: Some(Color::from([0.5803922, 0.92156863, 0.92156863]))
            }
        }
    }));
```

Alternating between the dark and light theme will then change the SVG accordingly.

## Implementation strategy

Add styling support to `iced_style`:

```rs
use iced_core::Color;

#[derive(Debug, Default, Clone, Copy)]
pub struct Appearance {
    pub fill: Option<Color>,
}

pub trait StyleSheet {
    type Style: Default + Copy;

    fn appearance(&self, style: Self::Style) -> Appearance;
}
```

Implement support for the built-in theme:

```
#[derive(Default, Clone, Copy)]
pub enum Svg {
    Custom(fn(&Theme) -> svg::Appearance),
    #[default]
    Default,
}

impl svg::StyleSheet for Theme {
    type Style = Svg;

    fn appearance(&self, style: Self::Style) -> svg::Appearance {
        match style {
            Svg::Default => Default::default(),
            Svg::Custom(appearance) => appearance(self),
        }
    }
}
```

Add an `iced_style::svg::Appearance` field to `svg::Handle`

```rs
/// Returns the styling [`Appearance`] for the SVG.
pub fn appearance(&self) -> Appearance {
    self.appearance
}

/// Set the [`Appearance`] for the SVG.
pub fn set_appearance(&mut self, appearance: Appearance) {
    self.appearance = appearance;
}
```

Add a `style: <Renderer::Theme as StyleSheet>::Style` field to `native::widget::svg` which can be defined with:

```rs
/// Sets the style variant of this [`Svg`].
pub fn style(
    mut self,
    style: <Renderer::Theme as StyleSheet>::Style,
) -> Self {
    self.style = style;
    self
}
```

Then within the `draw` method, grab the appearance from the theme:

```rs
let mut handle = self.handle.clone();
handle.set_appearance(theme.appearance(self.style));
renderer.draw(handle, drawing_bounds + offset);
```

Add a `into_rgb8` method to `iced_core::Color`:

```rs
/// Converts the [`Color`] into its RGBA8 equivalent.
pub fn into_rgba8(self) -> [u8; 4] {
    [
        (self.r * 255.0).round() as u8,
        (self.g * 255.0).round() as u8,
        (self.b * 255.0).round() as u8,
        (self.a * 255.0).round() as u8,
    ]
}
```

Within `wgpu::src/image/vector.rs`, apply the fill color to sufficiently non-transparent pixels:


```rs
let fill = handle.appearance().fill.map(crate::Color::into_rgba8);

...

rgba.chunks_exact_mut(4).for_each(|rgba| {
    if rgba[3] > 50 {
        if let Some(color) = fill {
            rgba[0] = color[0];
            rgba[1] = color[1];
            rgba[2] = color[2];
        }
    }

    rgba.swap(0, 2);
});
```

While being mindful to update the `Cache` to keep track of the appearance so that it is redrawn when the appearance changes.

## Drawbacks

It requires changing `widget::Svg` to `widget::Svg<Renderer>` to add the `style` field. Custom themes will also have to implement the new `svg::StyleSheet` trait.

## Rationale and alternatives

Packaging multiple colored variations of symbolic icons won't be practical. This solution applies the fill color at the time that the SVG is being rasterized, so icons can be in any color that the application developer or custom theme desires. This is able to leverage the advantages of the cache so that the fill does not have to be applied each time the widget is created.

The alternative is parsing the SVGs, traversing SVG nodes, and recreating a new SVG in-memory each time a svg widget is recreated. This would consume a large amount of CPU with the current way that the view is eagerly updated.

## Prior Art

CSS and GTK allow overriding the color of symbolic icons. Most commonly used to display a light variant of a symbolic icon against a dark background, or the default dark variant against a light background.

## Unresolved questions

None

## Future Possibilities

- The default theme could automatically choose a color based on the background it's to be rendered onto when the style is set to `Default`.
- More advanced forms of filtering could be implemented in the future. Such as darken and brighten.
- Perhaps it could be possible to pass a closure to the `Appearance` that would give access to mutating the rgba buffer.
- A similar strategy could be used for images to apply filters for them.
