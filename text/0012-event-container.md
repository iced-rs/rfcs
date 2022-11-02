# Event Container

## Summary

A programmable container that intercepts a variety of input events.

## Motivation

There are many widgets within a GUI library that need to handle a wider variety of input events than is capable from a button. From a left button press and release, to a right button press and release, middle button press and release, and mouse enter and exit.

## Guide-level explanation

### File icon example

One such use case for the event container is a file icon in a file manager. The left button press initiates a selection and drag, a left button release initiates a drop if the icon was dragged, a right button release opens a context menu, a middle click opens the file, a mouse enter causes the file icon to glow, and a mouse exit stops glowing the icon.

```rs
for (entity, metadata) in self.files.iter() {
    let file = file_icon(metadata)
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
    let tab = tab(metadata)
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
let headerbar = event_container(vec![left, center, right])
    .on_press(Message::WindowDrag)
    .on_release(Message::WindowMaximize)
    .on_right_release(Message::WindowContext);
```

## Implementation strategy

Implementation is very similar to the container widget, but with a handful of possible events that can be emitted:

```rs
struct EventMessages<Message> {
    on_press: Option<Message>,
    on_release: Option<Message>,
    on_right_press: Option<Message>,
    on_right_release: Option<Message>,
}
```

With an update method for publishing these messages on certain input events:

```rs
fn update<Message: Clone>(
    event: Event,
    layout: Layout<'_>,
    cursor_position: Point,
    shell: &mut Shell<'_, Message>,
    messages: &EventMessages<Message>,
) -> event::Status {
    let contains_bound = || layout.bounds().contains(cursor_position);

    if let Some(message) = messages.on_press.clone() {
        if let Event::Mouse(mouse::Event::ButtonPressed(mouse::Button::Left))
        | Event::Touch(touch::Event::FingerPressed { .. }) = event
        {
            return if contains_bound() {
                shell.publish(message);
                event::Status::Captured
            } else {
                event::Status::Ignored
            };
        }
    }

    if let Some(message) = messages.on_release.clone() {
        if let Event::Mouse(mouse::Event::ButtonReleased(mouse::Button::Left))
        | Event::Touch(touch::Event::FingerLifted { .. }) = event
        {
            return if contains_bound() {
                shell.publish(message);
                event::Status::Captured
            } else {
                event::Status::Ignored
            };
        }
    }

    if let Some(message) = messages.on_right_press.clone() {
        if let Event::Mouse(mouse::Event::ButtonPressed(mouse::Button::Right)) =
            event
        {
            return if contains_bound() {
                shell.publish(message);
                event::Status::Captured
            } else {
                event::Status::Ignored
            };
        }
    }

    if let Some(message) = messages.on_right_release.clone() {
        if let Event::Mouse(mouse::Event::ButtonReleased(
            mouse::Button::Right,
        )) = event
        {
            return if contains_bound() {
                shell.publish(message);
                event::Status::Captured
            } else {
                event::Status::Ignored
            };
        }
    }

    event::Status::Ignored
}
```

## Drawbacks

There's some duplicated code between the container widget and event_container widget.

## Rationale and alternatives

There doesn't seem to be any alternative approach besides enforcing application authors to bring their own event containers when needing to create a more sophisticated evented widget.

## Prior Art

Similar to `gtk::EventBox` from GTK, which is essentially a `gtk::Box` but with events that can be intercepted and programmed with callbacks. The implementation here is much more ergonomic in comparison.

## Unresolved questions

Perhaps there's a way to handle events in a more flexible way that isn't soo cumbersome, since directly matching input events is rather tedious.

## Future Possibilities

Enables a large number of more complex and featured widgets.