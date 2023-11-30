# Border Widths and Radiuses

## Summary

The ability to separately define the width of each side, and radius of each corner.

## Motivation

There are instances where a theme for a desktop GUI toolkit will require to specify a separate border width or radius. Such as a collection of conjoined buttons where only one side has a radius, or only the edges have a radius.

Designs for libcosmic have a navigation button within the left corner of the headerbar where the button hugs the edge of the window and defines a radius only the top-left and bottom-right corners. And there are some instances where a highlighted label likewise has a radius defined only for one side.

## Guide-level explanation

The border_width and border_radius are defined with a `Border`. Similar to `Padding`, this can be created from either a `f32`, `[f32; 2]`, or `[f32; 4]`. Internally, it is represented as:

```rs
pub struct Border([f32; 4]);
```

Which may be constructed with one of the following:

- `Border::from(10.0)`
- `Border::from([10.0, 5.0])`
- `Border::from([10.0, 5.0, 10.0, 0.0])`

For the `border_width` parameter, an input of `10.0` would assign a width of `10.0` to all sides. An input of `[10.0, 5.0]` would assign the top and bottom width to `10.0`, and the left and right width to `5.0`. Finally, an input of `[10.0, 5.0, 10.0, 0.0]` would assign widths clockwise starting from the top width of `10.0`, the right width of `5.0`, the bottom width of `10.0`, and the left width of `0.0`.

For the `border_radius` parameter, an input of `10.0` would assign a radius of `10.0` to all corners. An input of `[10.0, 5.0]` would assign the top-left and bottom-right radius to `10.0`, and the bottom-left and top-right radius to `5.0`. Finally, an input of `[10.0, 5.0, 10.0, 0.0]` would assign the radius clockwise starting from the top-left radius of `10.0`, the top-right radius of `5.0`, the bottom-right radius of `10.0`, and the bottom-left radius of `0.0`.


## Implementation strategy

- Create a `core::border` module containing a `Border` type that is identical to `Padding`.
- Update every instance of `border_radius` and `border_width` to use `Border`.
- Alter renderer to support the new conditional widths and radiuses.

## Drawbacks

Substantial changes required to convert every instance of `border_width` and `border_radius`. Any custom theme implementations would have to switch to the new type wherever they define these parameters.

## Rationale and alternatives

This space is already well explored by CSS. The border_width and border_radius parameters optionally accept one, two, three, or four parameters.

## Prior art

Both CSS and GTK support this functionality. GTK has container widgets which alter the radiuses of its children separately based on their position. Such as the `ButtonBox` container that defines a radius only for the left-most button and the right-most button, with all buttons inbetween having no radius.

## Unresolved questions

None

## Future possibilities

It would be possible to take inspiration from CSS and allow for the border to be defined as `Thin`, `Medium`, and `Thick`. Where the theme could define the values of these presets.
