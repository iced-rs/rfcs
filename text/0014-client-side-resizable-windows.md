# Client-side resizable windows

## Summary

The capability to resize the window with the mouse when client-side decorations are used.

## Motivation

Client-side decorations have become the norm for desktop applications, which requires that the GUI toolkit implements support for windowing features previously automatically provided by server-side decorations, such as the ability to resize the window with a mouse cursor drag from the edge of the window.

## Guide-level explanation

No explicit action would be required for the consumer of the toolkit, besides disabling decorations:

```rs
let mut settings = iced::Settings::default();
settings.window.min_size = Some((600, 300));
settings.window.decorations = false;
MyApp::run(settings)
```

A reasonable default border width can be chosen by Iced, such as `8`, otherwise the application can define it with

```rs
settings.window.border_size = 8;
```

## Implementation strategy

As client-side decorations will require that the application defines its own boundaries for handling mouse cursor resize events, `iced::window::Settings` will require a new `border_size` field. Which can be ignored when server-side decorations are enabled.

```rs
/// Size of the border resize handle when decorations are disabled.
pub border_size: u32,
```

Within `iced_winit::application::run_instance`, it will be necessary to track the resize direction throughout the lifetime of the instance. When a `winit::event::WindowEvent::CursorMoved` event is receievd in an application that lacks decorations and has a non-zero border-size, it can compare the cursor position against the size of the window to determine if the cursor is within the region of a resize zone. If the cursor has entered a new region, the icon of the cursor will be changed to reflect that.

```rs
let location = cursor_resize_direction(
    window.inner_size(),
    position,
    border_size as f64,
);
if location != cursor_prev_resize_direction {
    window.set_cursor_icon(cursor_direction_icon(
        location,
    ));
    cursor_prev_resize_direction = location;
    continue;
}
```

Meanwhile, when a `winit::event::WindowEvent::MouseInput` is received that is a left button press, the last-known state of the cursor can be checked to enable the `drag_resize_window` mode on the window. Which will cause the window to begin resizing to cursor movements until the left button is released.


```rs
if let Some(direction) = cursor_prev_resize_direction {
    let _res = window.drag_resize_window(direction);
    continue;
}
```

The cursor resize direction can be determined using this algorithm

```rs
n cursor_resize_direction(
    win_size: winit::dpi::PhysicalSize<u32>,
    position: winit::dpi::PhysicalPosition<f64>,
    border_size: f64,
) -> Option<ResizeDirection> {
    enum XDirection {
        West,
        East,
        Default,
    }

    enum YDirection {
        North,
        South,
        Default,
    }

    let xdir = if position.x < border_size {
        XDirection::West
    } else if position.x > (win_size.width as f64 - border_size) {
        XDirection::East
    } else {
        XDirection::Default
    };

    let ydir = if position.y < border_size {
        YDirection::North
    } else if position.y > (win_size.height as f64 - border_size) {
        YDirection::South
    } else {
        YDirection::Default
    };

    Some(match xdir {
        XDirection::West => match ydir {
            YDirection::North => ResizeDirection::NorthWest,
            YDirection::South => ResizeDirection::SouthWest,
            YDirection::Default => ResizeDirection::West,
        },

        XDirection::East => match ydir {
            YDirection::North => ResizeDirection::NorthEast,
            YDirection::South => ResizeDirection::SouthEast,
            YDirection::Default => ResizeDirection::East,
        },

        XDirection::Default => match ydir {
            YDirection::North => ResizeDirection::North,
            YDirection::South => ResizeDirection::South,
            YDirection::Default => return None,
        },
    })
}
```

## Drawbacks

Platform support is currently limited. I have been in the process of getting changes merged for X11. Wayland will require wayland-rs to get a non-beta release. Windows support can be implemented the moment the X11 PR is merged. And Mac OS enforces resizable borders even if decorations are disabled, so this functionality is not required for Mac OS. It's also not required for Android/iOS or the web.

Patches to the iced-rs fork of winit will be required to enable this functionality for X11 today.

## Rationale and alternatives

This enables support for client-side window resize in a way that is not invasive to the consumer. It's not viable to implement this in any other way, and lack of this will render client-side window decorations useless on Linux.

## Unresolved questions

None

## Future Possibilities

None