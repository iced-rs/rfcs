# Generic Command Buffers

Also see PR from which I've migrated most of this text, which includes an
example implementation: <https://github.com/iced-rs/iced/pull/1620>.

## Summary

This proposes to change how commands are passed out of applications by receiving
commands through a generic `impl Commands<Self::Message>` buffer instead of
having the constructor and update functions return a `Command<Self::Message>`:

```rust
impl Application for MyAwesomeApplication {
    type Message = Message;
    /* ... */

    fn new(flags: Self::Flags, commands: impl Commands<Self::Message>) -> Self {
        /* ... */
    }

    fn update(message: Self::Message, commands: impl Commands<Self::Message>) {
        /* ... */
    }
}
```


## Motivation

Returning commands have several deficiencies:

* Returning a batch requires the allocation of a vector, composing batches from
  sub-components can then easily result in many vectors of vectors being
  allocated and operated over.
* Composing commands from sub-components require more indirection and
  allocations than are necessary, especially when dealing with futures which are
  always boxed.

The proposed API would allow for implementing additional allocation improvements
such as:
* Re-using allocations for spawned futures using a similar stragey as the one in
  [`reusable-box-futures`](https://docs.rs/reusable-box-future) built on top of
  a slab allocator.
* Not allocating certain things on the heap at all, like a command buffer which
  uses stack space can be used with iced applications.
* Coalescing commands earlier if appropriate. For example closing or resizing
  the window only needs to retain the latest command issued.


## Guide-level explanation

This proposes to introduce the following trait and associated types to use as
shown above ([see my branch for proposed documentation and
tests](https://github.com/udoprog/iced/blob/commands/native/src/commands.rs)):

```rust
pub trait Commands<T> {
    type ByRef<'a>: Commands<T>
    where
        Self: 'a;

    fn by_ref(&mut self) -> Self::ByRef<'_>;

    fn perform<F>(
        &mut self,
        future: F,
        map: impl Fn(F::Output) -> T + MaybeSend + Sync + 'static,
    ) where
        F: Future + 'static + MaybeSend;

    fn command(&mut self, command: Command<T>);

    fn extend<I>(&mut self, iter: I)
    where
        I: IntoIterator<Item = Command<T>>
    {
        /* ... */
    }

    fn map<M, U>(self, map: M) -> Map<Self, M>
    where
        Self: Sized,
        M: MaybeSend + Sync + Clone + Fn(U) -> T
    {
        /* ... */
    }
}

#[derive(Debug)]
pub struct Map<C, M> {
    /* ... */
}

impl<T: 'static, C, M: 'static, U: 'static> Commands<U> for Map<C, M>
where
    C: Commands<T>,
    M: MaybeSend + Sync + Clone + Fn(U) -> T
{
    /* ... */
}
```

`Command` would be changed into the following and most of its existing plumbing
dropped, note particularly how the `Future` variant is no longer necessary since
it can be received directly by the command buffer:

```rust
pub enum Command<T> {
    Clipboard(clipboard::Action<T>),
    Window(window::Action<T>),
    System(system::Action<T>),
    Widget(widget::Action<T>),
}
```


#### Motivating `Commands::perform`:

Previously when components were composed, futures where boxed early to be
shipped in a `Command`. So [mapping over them means mapping over an already
boxed
future](https://github.com/iced-rs/iced/blob/11f5527d7645619f49b030e30485f24ac637efbd/native/src/command/action.rs#L47).

If we instead pass down the `map` function `perform` the underlying command
buffer can do the transformation once, effectively reducing the number of
allocations and pointer indirection used to a single one.

You can see that in action [in my
branch](https://github.com/udoprog/iced/blob/27e40c93635ff48451b5df9e6b65834701fbc863/native/src/commands.rs#L228).
Note how the future is passed through and all the message mapping happens
directly in the callback function rather than inside of a boxed future.


#### Motivating `Commands::by_ref`:

The naive approach would be to simply use `&mut commands` whenever you want to
reborrow a command buffer which actually works since we provide a `&mut C where
C: Commands<T>` implementation.

But this has the following downsides:

* Nesting components can cause complex types to be generated, such as `&mut &mut
  &mut &mut playground::Foo` [in this
  playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=442ff812cbffa1dddbb376ccf9d73936).
  In generic code in particular this can lead to problems such as very large
  type signatures or even be flat out rejected by rustc for the type being too
  large ([note the difference with
  `Trait::by_ref`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=47a54551e72ffa86702fdf91980a414c)).
* `commands.by_ref().map(/*  */)` has in my mind better ergonomics than `(&mut
  commands).map(/*  */)`, see also
  [`Iterator::by_ref`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.by_ref)
  so even if we remove the GAT it's in my mind worth keeping.


#### Motivating `Commands::map`:

As show in this test case, this (in combination with `by_ref`) is how the
feature interacts with sub-components compositionally:

```rust
use iced::Commands;

enum Message {
    Component1(component1::Message),
    Component2(component2::Message),
}

fn update(mut commands: impl Commands<Message>) {
    component1::update(commands.by_ref().map(Message::Component1));
    component2::update(commands.by_ref().map(Message::Component2));
}

mod component1 {
    use iced::Commands;

    pub(crate) enum Message {
        Tick,
    }

    pub(crate) fn update(mut commands: impl Commands<Message>) {
        /* ... */
    }
}

mod component2 {
    use iced::Commands;

    pub(crate) enum Message {
        Tick,
    }

    pub(crate) fn update(mut commands: impl Commands<Message>) {
        /* ... */
    }
}
```

## Implementation strategy

There isn't much too the implementation strategy. Do and advertise the change.

See this PR for a working implementation:
<https://github.com/iced-rs/iced/pull/1620>.

See this application and branch for example use:
<https://github.com/udoprog/ontv/tree/commands>.


## Drawbacks

Commands are no longer returned, which is a change. However this does in many
instances lead to much cleaner code with *less* `Command::none()` padding.
