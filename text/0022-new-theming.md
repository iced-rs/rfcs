# New theme system

## Summary

The idea is to get rid of Style Sheet and simplify the custom widget implementation.

- the library will take a color sheme (Light and Dark), that we define at launch.
- we can tell the library if it should use the light or dark theme
- we don't have to specifie the theme, it should have default themes

## Motivation

Because right now, if we want for example, 2 containers with 2 differents colors, we need to create our own ContainerType, and then implement `container::StyleSheet` for our custom theme.
This bring a lot of new concept for new users.


## Guide-level explanation

Iced will have a type Palette:
```
/// A color palette type interne to the library
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Palette {
    /// The background [`Color`] of the [`Palette`].
    pub background: Color,
    /// The text [`Color`] of the [`Palette`].
    pub text: Color,
    /// The primary [`Color`] of the [`Palette`].
    pub primary: Color,
    /// The success [`Color`] of the [`Palette`].
    pub success: Color,
    /// The danger [`Color`] of the [`Palette`].
    pub danger: Color,
}
```
When we start the program, we can give our own implementation of DarkPalette and LightPalette to Iced.
We also need a mechanism to tell iced when to use DarkPalette or LightPalette.

There will be no custom theme. The apparence of the widgets will be created from the shemes colors.

If, we wan't more than 2 themes, or if we need more customisation (for example, if we need 2 container with 2 differents colors):
- we could define our own colors sheme to fit our need:

```
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct CustomPalette {
    pub primary: Color,
    pub onPrimary: Color,
    pub thertiary: Color,
    pub onTertiary: Color,
    pub secondary: Color,
    pub onSecondary: Color,
    pub background: Color,
    pub text: Color,
    pub primary: Color,
    pub success: Color,
    pub danger: Color,
}
```
And then we could just overide properties like that:
```
Container::new()
    .background(self.theme.primary)
    .border_color(self.theme.onPrimary)
    .border_width(2f32)
```

Notice we could set border_width, that way, we don't need style sheet anymore.

If we want to provide different style of Container, we could simply give an enumetation to `new()`

```
Container::new(iced::Container::Boxed)
```

## Implementation strategy

Just define more method for each widget. The default attributes will be set according to the theme of the application.
The style parameter will be removed and replaced by the self attributes of the structure in question.

## Drawbacks

- it would break the api
- maybe other things that I'm not aware of (I'm new to Rust)

## Rationale and alternatives

If we continue with the current style system, we force ourself to make the implementation of a widget style sheet in one place.
With this approach, we could easily make a custom view function with whatever widget we wan't, and it will have no effect with other fonctionnaly that we have declared in other files.
Also, it will make small scale customization easier for new users.
This brings quick widget configuration, while leaving the possibility of creating your own widget if you really want a lot of customization.

## [Optional] Prior art

- Does this feature exist in other GUI toolkits and what experience have their community had?

All my ideas come from Jetpack Compose. That's exactly how it manages customization and I find it easier, with less boiler plate.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

I have already done a test with Container and I am able to change the background color

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## [Optional] Future possibilities


