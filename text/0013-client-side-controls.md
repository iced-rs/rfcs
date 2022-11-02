# Client-side window controls

## Summary

Enables applications with client-side decorations to maximize, minimize, and initiate a window drag with the mouse.

## Motivation

GUI toolkits implementing client-side decorations need to implement a headerbar widget which has access to triggering window methods to maximize, minimize, and drag the window with the mouse.

## Guide-level explanation

An application will be able define messages for window close, drag, minimize, and maximize, then call upon these window commands from the update method.

```rs
#[derive(Debug, Clone)]
pub enum Message {
    Close,
    Drag,
    Minimize,
    Maximize,
}

fn update(&mut self, message: Message) -> iced::Command<Self::Message> {
    match message 
        Message::Close => self.should_exit = true,
        Message::Drag => return iced_native::window::drag(),
        Message::Minimize => return iced_native::window::minimize(),
        Message::Maximize => return iced_native::window::toggle_maximize(),
        ...
    }
    
    Command::none()
}
```

A headerbar widget can be utilized or implemented that passes these events to the applications.


## Implementation strategy

Requires adding the following new variants to `native::window::action::Action`:

- `Drag`
- `Maximize(bool)`,
- `Minimize(bool)`,

It's also possible to implement a convenience method for toggling maximization:

- `ToggleMaximize`

Then `iced_winit::application::run_command` can check for these methods and call the respective winit methods.

## Drawbacks

A headerbar widget is not currently provided by iced, so it is required for applications to create their own to trigger these events. This also requires using the `Application` trait rather than `Sandbox` to take advantage of commands.

## Rationale and alternatives

There already exists an enum for handling window actions such as these, and functions for generating commands, so it would make sense to implement the support in this way. The drag event is crucial to handle the moment the event is received so that the drag event can immediately begin.

An alternative approach would be expanding the `Application` and `Sandbox` traits to provide `should_maximize`, `should_minimize`, and `should_drag` methods; which would be similar to the existing `should_exit` method.

## Unresolved questions

None

## Future Possibilities

None
