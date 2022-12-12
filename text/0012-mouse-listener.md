# Mouse Listener

## Summary

A programmable container that intercepts a variety of mouse input events.

## Motivation

There are many widgets within a GUI library that need to handle a wider variety of input events than is capable from a button. From a left button press and release, to a right button press and release, middle button press and release, and mouse enter and exit.

## Guide-level explanation

### File icon example

One such use case for the event container is a file icon in a file manager. The left button press initiates a selection and drag, a left button release initiates a drop if the icon was dragged, a right button release opens a context menu, a middle click opens the file, a mouse enter causes the file icon to glow, and a mouse exit stops glowing the icon.

```rs
for (entity, metadata) in self.files.iter() {
    let file = mouse_listener(file_icon(metadata))
        .on_press(Message::Drag(entity))
        .on_release(Message::Drop)
        .on_right_release(Message::Context(entity))
        .on_middle_release(Message::OpenFile(entity))
        .on_mouse_enter(Message::IconEnter(entity))
        .on_mouse_exit(Message::IconExit(entity));

    icon_widgets.push(file.into());
}
```

### Tabs

A tabbed interface may also want to provide widgets that are draggable with context on a right click:

```rs
for (entity, metadata) in self.tabs.iter() {
    let tab = mouse_listener(tab(metadata))
        .on_press(Message::TabDrag(entity))
        .on_release(Message::TabDrop)
        .on_right_release(Message::TabContext(entity))
        .on_middle_release(Message::TabClose(entity))
        .on_mouse_enter(Message::TabHighlight(entity))
        .on_mouse_exit(Message::TabUnhighlight(entity));

    tabs.push(tab);
}
```

### Headerbar example

Another use case for this widget is a headerbar, which is a container placed at the top of the window to serve as a replacement for a window title bar when server-side decorations are disabled. If any part of the container is clicked that does not intercept that click event, then the container itself will receive the click and make it possible to initiate a window drag.

```rs
let headerbar = mouse_listener(row(vec![left, center, right]))
    .on_press(Message::WindowDrag)
    .on_release(Message::WindowMaximize)
    .on_right_release(Message::WindowContext);
```

## Implementation strategy

The implementation is simple, and only requires thinly wrapping any existing element to intercept mouse events over it.

```rs
pub struct MouseListener<'a, Message, Renderer> {
    content: Element<'a, Message, Renderer>,

    /// Sets the message to emit on a left mouse button press.
    on_press: Option<Message>,

    /// Sets the message to emit on a left mouse button release.
    on_release: Option<Message>,

    /// Sets the message to emit on a right mouse button press.
    on_right_press: Option<Message>,

    /// Sets the message to emit on a right mouse button release.
    on_right_release: Option<Message>,

    /// Sets the message to emit on a middle mouse button press.
    on_middle_press: Option<Message>,

    /// Sets the message to emit on a middle mouse button release.
    on_middle_release: Option<Message>,

    /// Sets the message to emit when the mouse enters the widget.
    on_mouse_enter: Option<Message>,

    /// Sets the messsage to emit when the mouse exits the widget.
    on_mouse_exit: Option<Message>,
}
```

The update method will emit these messages when their criteria is met.

## Drawbacks

N/A

## Rationale and alternatives

The alternative is that it'd be required to manually implement the Widget trait whenever you want to have a widget intercept a variety of mouse events outside of the left button release event that the button widget currently provides. This will significantly improve ergonomics for creating more sophisticated widgets.

## Prior Art

Similar to `gtk::EventBox` from GTK, which is essentially a `gtk::Box` but with events that can be intercepted and programmed with callbacks. The implementation here is much more ergonomic in comparison.

## Unresolved questions

None

## Future Possibilities

Enables a large number of more complex and featured widgets.