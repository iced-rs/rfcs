# Multi-windowed application

## Summary

This proposal aims to introduce the concept of multi-windowed applications to iced, while conforming as best as possible to its current architecture and goals.
One of the main goals of this proposal is to keep the control of multiple windows simple and consistent with the rest of iced's architecture.

## Motivation

There are certain applications that might need or would benefit from the use of multiple windows.

Following the discussion from [#27](https://github.com/iced-rs/iced/issues/27), currently there is no way to create or manipulate multiple windows in iced.
This proposal once implemented would allow the creation and manipulation of multiple windows within a single application.

## Guide-level explanation

This proposal starts by introducing a new `multi_window::Application` trait, while it's mostly the same as the current `Application` trait,
there are some main changes.

### Viewing multiple windows

The biggest change in `multi_window::Application` is it's `view` method. The goal of this proposal is allowing a multi-windowed application in iced,
to do that we to introduce the concept of multiple `view` calls. For that, we change the `view` method as shown:  

```rust
impl multi_window::Application for Example{
    // ...

    fn view(&self, window: window::Id) -> Element<Message>{
        // ...
    }
}
```

This new `view` method, has a new `window` parameter of `window::Id` type. This new type is better explained later, right now you can see it as a way to unique tag to your window.
By attaching a unique identifier to a window, the runtime can call `view` with this identifier whenever it needs to paint that window.

**Note:** This behaviour should be 'shell' specific and not relied upon, optimizations may happen between versions!

As a way to better understand how this new method could be used, take these two examples:

#### **Fixed number of windows**

One of the ways to use this method, is to have a compile-time known number of windows.
```rust
const MAIN_WINDOW: window::Id = window::Id::new(0);
const DEBUG_WINDOW: window::Id = window::Id::new(1);

impl multi_window::Application for Example{
    // ...

    fn view(&self, window: window::Id) -> Element<Message>{
        match window{
            MAIN_WINDOW => /*...*/,
            DEBUG_WINDOW => /*...*/,
        }
    }
}
```

#### **Dynamic number of windows**

But most applications will want to use a dynamic number of windows.
```rust
impl multi_window::Application for Example{
    // ...

    fn view(&self, window: window::Id) -> Element<Message>{
        if let Some(window) = self.windows.get(&window){
            // ...
        }

        // ...
    }
}
```

### Creating a window

Before you can draw to a window, first you need to spawn it!
To create a new window, the new `Application::windows` method should be used:
```rust
struct Example;

impl multi_window::Application for Example{
    // ...

    fn windows(&self) -> Vec<(window::Id, window::Settings)>{
        // ...
    }
}
```

You can think of this method as something similar to `Application::title`, the runtime will constantly check if you returned
something diferent and then make changes based on that!

More specifically, the runtime will check if there any changes in the list, and then make changes to the windows based on that.
You should be able to:
- Create a window, by adding a new entry to the list
- Close a window, by removing an entry from the list
- Modify a window, by editing an entry on the list

#### **Fixed number of windows**
It's possible to have a compile-time known list of windows.

```rust
const MAIN_WINDOW: window::Id = window::Id::new(0);
const DEBUG_WINDOW: window::Id = window::Id::new(1);

impl multi_window::Application for Example{
    // ...

    fn windows(&self) -> Vec<(window::Id, window::Settings)>{
        vec![(MAIN_WINDOW, window::Settings::default()), (DEBUG_WINDOW, window::Settings::default())]
    }
}
```

#### **Dynamic number of windows**
But most applications will want to use a dynamic list of windows.

```rust
struct Example{
    windows: HashMap<window::Id, window::Settings>,
    // ...
}

impl multi_window::Application for Example{
    // ...

    fn windows(&self) -> Vec<(window::Id, window::Settings)>{
        self.windows.collect()
    }
}
```

### Closing a window

One of the main ways to interact with a window, is requesting for it to be closed.
This can be done in multiple ways, but the fact is that when this happens the user expects some feedback.
To address this, a new method is introduced `Application::close_requested`:
```rust
impl multi_window::Application for Example{
    // ...

    fn close_requested(&self, window: window::Id) -> Self::Message {
        // ...
    }
}
```

This method is called whenever the user in anyway possible, requests a window to be closed, which is identified by the `window` parameter.
**Note:** You can check the the [implementation strategy](#implementation-strategy) section on why this has been introduced.
```rust
enum Message{
    WindowClosed(window::Id),
}

impl multi_window::Application for Example{
    // ...

    fn update(&mut self, message: Self::Message) -> Command<Self::Message>{
        match message{
            Message::CloseRequested(window) => {
                self.windows.remove(&window);
            }
        }
    }

    fn close_requested(&self, window: window::Id) -> Self::Message {
        Message::CloseRequested(window)
    }
}
```

## Implementation strategy

**Note:** As of writing this RFC, a prototype for the changes described here is already done, it contains the initial changes that would allow a simple multi-windowed application. You can check the work-in-progress branch [here](https://github.com/derezzedex/iced/tree/feat/multi-window).

The implentation strategy used in the prototype is heavily dependent on some quirks of the current `iced_native` implementation, but the multi-windowed 'interface' that has been described
in the previous section could be implemented in multiple ways.

One of the major goals of iced is keeping it's ecosystem modular, but it still has some modules and structures that are central to its working.
The biggest implementation point of iced (or more specifically `iced_native`) is the `UserInterface`, this struct keeps a list of widgets and is used to update and draw them.
The `UserInterface` members are somewhat simple, you have the main components necessary to update, layout and draw every widget that is contained in the bounds of the window.

```rust
pub struct UserInterface<'a, Message, Renderer> {
    // This is the root `Widget` of your application,
    // the `UserInterface` also recursively walks through its children.
    root: Element<'a, Message, Renderer>,

    // This the 'bounds' of the root element
    // and its children, you can consider this
    // the 'layout' of your application.
    base: layout::Node,
    // This is basically the same as the `base`,
    // but used as an `overlay` (no layering yet!).
    overlay: Option<layout::Node>,

    // This is the size of the viewport,
    // the bounds of the application.
    bounds: Size,
}
```
The `UserInterface` is what makes the core of the application loop in `iced_native`, it's usage is similar to the following small example.
```rust
loop {
    // Process system events here...

    // We call the `Application::view` method to get the widgets
    let root = application.view();

    // And then use them to build the `UserInterface`
    let user_interface = UserInterface::build(
        &application,
        root,
        // ...        
    );

    // Update the user interface
    user_interface.update(/* ... */);
    // And then draw it
    user_interface.draw(/* ... */);
}
```

Hopefully now you have an understanding of how the `UserInteface` is very important and that it would then make sense, that in the implementation of multi-window support, this would also be where the biggest change is: now we need to have multiple `UserInterface`s.

Although the change is much more complex than this, the main idea is that we now have a list of `UserInterface`, something similar to:
```rust
let mut user_intefaces: HashMap<window::Id, UserInterface> = HashMap::new();

loop {
    for (id, window) in application.windows(){
        // Process window specific events here...

        // We call the `Application::view` method to get the widgets
        let root = application.view(id);
    
        // And then use them to build the `UserInterface`
        let user_interface = user_interfaces.get_mut(&id);

        // Update the user interface
        user_interface.update(/* ... */);
        // And then draw it
        user_interface.draw(/* ... */);
    }
}
```

And now, instead of calling `build_user_interface` directly, we use it in a new helper function that returns a list:
```rust
fn build_user_interfaces<'a, A, C>(
    application: &'a A,
    renderer: &mut A::Renderer,
    debug: &mut Debug,
    states: &HashMap<window::Id, WindowState<A, C>>,
    mut pure_states: HashMap<window::Id, iced_pure::State>,
) -> HashMap<
    window::Id,
    UserInterface<
        'a,
        <A as Application>::Message,
        <A as Application>::Renderer,
    >,
>
where
    A: Application + 'static,
    C: iced_graphics::window::Compositor<Renderer = A::Renderer> + 'static,
{
    let mut interfaces = HashMap::new();

    for (id, pure_state) in pure_states.drain() {
        let state = &states.get(&id).unwrap().state;

        let user_interface = build_user_interface(
            application,
            user_interface::Cache::default(),
            renderer,
            state.logical_size(),
            debug,
            pure_state,
            id,
        );

        let _ = interfaces.insert(id, user_interface);
    }

    interfaces
}
```

Not only that, but we also need to change the way we use the `application::State` struct.
We now use multiple `State` structs, by attaching them with their own window `Surface`.

```rust
struct WindowState<
    A: Application,
    C: iced_graphics::window::Compositor<Renderer = A::Renderer>,
> {
    surface: <C as iced_graphics::window::Compositor>::Surface,
    state: State<A>,
}
```

Unfortunately, we need some additional changes. The way the current implentation works, we need to introduce a new `Event` wrapper to separate the user `Message` with our own internal `Message`s!

The biggest reason we do this, is because `winit` has control of the `event_loop` and we can't send it to the future that controls the application's main loop.
To work around that, we divide the event of spawning a new window in two parts: actual window creation and then, window 'integration'.

```rust
// This is an wrapper around the `Application::Message` associate type
// that allows the `shell` to create internal messages, while still having
// the current user created messages.
#[derive(Debug)]
pub enum Event<Message> {
    // This contains the `Application::Message`,
    // allowing it to work as it did previously.
    Application(Message),

    // These are the new internal `Event`s,
    // they make it possible to send and receive
    // data to the main thread.
    NewWindow(window::Id, settings::Window),
    CloseWindow(window::Id),
    WindowCreated(window::Id, winit::window::Window),
}
```

The actual window creation happens in the 'main thread', by matching and then mapping the `winit::Event` early:
```rust
winit::event::Event::UserEvent(Event::NewWindow(id, settings)) => {
    let window = settings
        .clone()
        .into_builder(
            settings.title,
            Mode::Windowed,
            event_loop.primary_monitor(),
            None,
        )
        .build(&event_loop)
        .expect("Failed to build window");

    Some(winit::event::Event::UserEvent(Event::WindowCreated(
        id, window,
    )))
}
```

Later we can match on this `Event`, and use it to properly introduce the rendering and framework context:
```rust
Event::WindowCreated(id, window) => {
    let mut surface = compositor.create_surface(&window);

    let state = State::new(&application, &window);

    let physical_size = state.physical_size();

    compositor.configure_surface(
        &mut surface,
        physical_size.width,
        physical_size.height,
    );

    let pure_state = iced_pure::State::new();
    let user_interface = build_user_interface(
        &application,
        user_interface::Cache::default(),
        &mut renderer,
        state.logical_size(),
        &mut debug,
        pure_state,
        id,
    );

    let window_state: WindowState<A, C> =
        WindowState { surface, state };

    // To make it easier to create a prototype,
    // multiple lists were made instead of only one
    // (that would require changing a lot more code).
    let _ = states.insert(id, window_state);
    let _ = interfaces.insert(id, user_interface);
    let _ = window_ids.insert(window.id(), id);
    let _ = windows.insert(id, window);
}
```

And for the the user events, we can just do as we did before:
```rust
Event::Application(message) => {
    messages.push(message);
}
```

And for the actual window spawning, closing and manipulation we use the `State::synchronize` to do that:
```rust
pub fn synchronize(
    &mut self,
    application: &A,
    windows: &HashMap<window::Id, Window>,
    proxy: &EventLoopProxy<Event<A::Message>>,
) {
    let new_windows = application.windows();

    // Check for windows to close
    for window_id in windows.keys() {
        if !new_windows.iter().any(|(id, _)| id == window_id) {
            proxy
                .send_event(Event::CloseWindow(*window_id))
                .expect("Failed to send message");
        }
    }

    // Check for windows to spawn
    for (id, settings) in new_windows {
        if !windows.contains_key(&id) {
            proxy
                .send_event(Event::NewWindow(id, settings))
                .expect("Failed to send message");
        }
    }
    
    //...
}
```

## Drawbacks

By introducing another version of the `Application` trait, the drawbacks of this proposal are exclusive to it's usage.
Currently, the main drawback of this is the decreased performance that this could cause. The main reason for this is
that there isn't a way to know which window should be redrawn and renderered, meaning that each time an update occurs,
we need to redraw every window.

## Rationale and alternatives

Very early on there was another idea on the `multi_window::Application`, the "multiple applications" design.
This would make each window be it's own `Application`. By forcing the user to specify a new type of 'message' the `Application::External`, essentialy having a new list of possible 'messages' that the application could receive from other applications/windows.

The idea was scraped because it would require require a lot more changes, being much harder to implement while
not being as easy to use as having a `window::Id` in `Application::view` (thank you @hecrj for the idea).

## Unresolved questions

- Should we remove the `window::Settings` from the application `settings`?
- Is there a way to optimize the current implementation, improving performance?
- Is it possible to remove the need of returning a `Vec` in `Application::windows`?
- Should we remove `Application::title` to `window::Settings` or just add `window::Id` as a parameter.

## Future possibilities

- Window 'drag and drop' support

You can check a prototype example of this in action at [derezzedex/iced](https://github.com/derezzedex/iced/tree/feat/multi-window), note that this is a work-in-progress starting point, so there are a couple of missing things!
