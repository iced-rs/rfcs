# Accessibility

## Summary

This proposal aims to introduce Accessibility support to Iced without changing the existing architecture. Support for each widget can be added incrementally after the iced_accessibility crate and other components are added.


## Motivation

Accessibility support is an important feature for GUI toolkits. [Accesskit](https://github.com/AccessKit/accesskit) provides a cross-platform interface for interacting with accessibility software. It is still a work in progress, but already supports many widget types, and is used in egui. Accesibility support is currently missing in Iced, but could be implemented using Accesskit and the strategies in this proposal.


## Guide-level explanation

This proposal is transparent to most Iced developers. Only widget implementors need to interact with the new method introduced in the `Widget` trait for emitting the accessibility nodes for their widget, if they choose to make it accessible. Accessibility support for each widget is optional, and can be added incrementally.

Some initial implementations using this method are [here for the button](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/button.rs#L291), [here for the container](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/container.rs#L269), [here for a row](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/row.rs#L247) and [here for Text](https://github.com/wash2/iced/blob/45780fc8a764ea0075b4139a163c13b0a0aad0a8/native/src/widget/text.rs#L176). While the current implementations may not be complete, they demonstrate usage.


## Implementation strategy


This proposal starts with the `iced_accessibility` crate and the types that it introduces. `iced_accessibility` re-exports the accesskit crate and the chosen feature-gated platform adapter, one of `accesskit-winit` (most likely), `accesskit-unix`, `accesskit-windows`, or `accesskit-macos`. Iced-rs would likely want to maintain a fork of accesskit that sets the winit dependency of `accesskit-winit` to https://github.com/iced-rs/winit. 

`iced_accessibility` also provides the following types for use in `iced_native`, `iced_winit`, `iced_glutin`, or other iced crates that need to build accessibility trees from accesskit nodes.

### A11yId

Accessibility Nodes need relatively consistent Ids for each element in the tree. A11yId is built using a slightly modified version of the existing `iced_native::widget::id::Id`. Because `iced_native` will depend on `iced_accessibility` the `Id` type is moved to `iced_core` and be re-exported by `iced_native`.

Iced Widget `Id` implementation previously used usize, but now it will use u64 for widgets, and also supports creating `Id`s for windows, which also need a relatively consistent NodeId. `accesskit::NodeId` is backed by non-zero `u128`, so Widget `Id`s occupy the range `[1, u64::MAX]`, while window `Id`'s occupy `[u64::MAX + 1, u128::MAX]`. This avoids collisions between the two types of `Id`s, which are generated without knowledge of eachother in somewhat different ways. See an initial implementation [here](https://github.com/wash2/iced/blob/a11y/core/src/id.rs).

[`A11yId`](https://github.com/wash2/iced/blob/a11y/accessibility/src/id.rs) wraps the different `Id`s and implements conversion between Iced Id types and accesskit NodeId types for convenient usage:
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum A11yId {
    Window(NonZeroU128),
    Widget(Id),
}
```

### A11yNode

[`A11yNode`](https://github.com/wash2/iced/blob/a11y/accessibility/src/node.rs) wraps the previously defined `A11yId` and an `accesskit:Node`. It implements helpers for constructing a node from an Iced Id and a node, adding children, and conversion to the expected accesskit tuple format. 

```rust
#[derive(Debug, Clone)]
pub struct A11yNode {
    node: accesskit::Node,
    id: A11yId,
}
```

### A11yTree

[A11yTree](https://github.com/wash2/iced/blob/a11y/accessibility/src/a11y_tree.rs) is a tree abstraction around the accesskit Node lists that are used to represent the accessibility tree. It includes helpers for building a tree for common types of Widgets, though the list of helpers could probably be expanded in the future. What is currently proposed are:
- a helper for creating an A11yTree for a widget with a single root node and some children
- a helper for merging the trees of multiple children, primarily used in widgets like `Row` and `Column` which don't want to create an accessibility node themselves, but want to expose their children's node to their parent widget.
- a helper for creating an A11yTree for a leaf node. a single node with no children

Lastly, the tree can also be built directly from a list of root `A11yNode`s and child `A11yNode`s to accomodate widgets that don't fit in neatly to the above categories. `A11yTree` also implements conversion to the accesskit `Vec<(NodeId, Node)>` for sending tree updates.

```rust
#[derive(Debug, Clone, Default)]
/// Accessible tree of nodes
pub struct A11yTree {
    /// The root of the current widget, children of the parent widget or the Window if there is no parent widget
    root: Vec<A11yNode>,
    /// The children of a widget and its children
    children: Vec<A11yNode>,
}
```

### Additions to iced_native

This proposal requires changes to iced_native in addition to moving the `Id` implementation. `Accessibility` additions are (mostly) behind a feature gate. They include:
- New optional method for the `Widget` trait: `a11y_nodes`
- Wrapper types for Iced `Operation` & usage of the wrapper type internally. This is required for using Operations internally in `iced_winit` or other shells to retreive application focus. This is a change that is not feature gated, but is transparent to users.
- addition of `a11y_nodes` method to `UserInterface`
- addition of `Event::A11y` variant

### New optional method for the `Widget` trait: `a11y_nodes`

The default implementation allows widgets to opt-in to supporting accessibility for themselves and their children. It uses the previously defined `iced_accessibility` traits for convenience.
```rust    
#[cfg(feature = "a11y")]
/// get the a11y nodes for the widget and its children
fn a11y_nodes(&self, _layout: Layout<'_>) -> iced_accessibility::A11yTree {
    iced_accessibility::A11yTree::default()
}
```

Some initial implementations using this method are [here for the button](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/button.rs#L291), [here for the container](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/container.rs#L269), [here for a row](https://github.com/wash2/iced/blob/c260dd413b2aeebdd79a8ce926aa9fc5556f048b/native/src/widget/row.rs#L247) and [here for Text](https://github.com/wash2/iced/blob/45780fc8a764ea0075b4139a163c13b0a0aad0a8/native/src/widget/text.rs#L176). While the current implementations may not be complete, they demonstrate usage.

### Wrapper types for Iced `Operation` & usage of the wrapper type internally

Operation must support `Application::Message` type parameter to be used by iced applications as it currently exists. Operations are also the currently preferred method for retreiving application focus, so they must be used to access the focus for communicating widget focus to accesskit. `OperationWrapper` and `OperationWrapperOutput` make this possible. It requires a small change to the `operate` signature in UserInterface, Widget, Overlay, and element types.

```rust
pub fn operate(
        &mut self,
        renderer: &Renderer,
        operation: &mut dyn widget::Operation<OperationOutputWrapper<Message>>,
    ) {...}
```

```rust
#[allow(missing_debug_implementations)]
/// A wrapper around an [`Operation`] that can be used for Application Messages and internally in Iced.
pub enum OperationWrapper<M> {
    /// Application Message
    Message(Box<dyn Operation<M>>),
    /// Widget Id
    Id(Box<dyn Operation<crate::widget::Id>>),
    /// Wrapper
    Wrapper(Box<dyn Operation<OperationOutputWrapper<M>>>),
}

#[allow(missing_debug_implementations)]
/// A wrapper around an [`Operation`] output that can be used for Application Messages and internally in Iced.
pub enum OperationOutputWrapper<M> {
    /// Application Message
    Message(M),
    /// Widget Id
    Id(crate::widget::Id),
}

impl<M: 'static> Operation<OperationOutputWrapper<M>> for OperationWrapper<M> {
    fn container(
        &mut self,
        id: Option<&Id>,
        operate_on_children: &mut dyn FnMut(
            &mut dyn Operation<OperationOutputWrapper<M>>,
        ),
    ) {
        match self {
            OperationWrapper::Message(operation) => {
                operation.container(id, &mut |operation| {
                    operate_on_children(&mut MapOperation { operation });
                });
            }
            OperationWrapper::Id(operation) => {
                operation.container(id, &mut |operation| {
                    operate_on_children(&mut MapOperation { operation });
                });
            }
            OperationWrapper::Wrapper(operation) => {
                operation.container(id, operate_on_children);
            }
        }
    }

    fn focusable(&mut self, state: &mut dyn Focusable, id: Option<&Id>) {
        match self {
            OperationWrapper::Message(operation) => {
                operation.focusable(state, id);
            }
            OperationWrapper::Id(operation) => {
                operation.focusable(state, id);
            }
            OperationWrapper::Wrapper(operation) => {
                operation.focusable(state, id);
            }
        }
    }

    fn scrollable(&mut self, state: &mut dyn Scrollable, id: Option<&Id>) {
        match self {
            OperationWrapper::Message(operation) => {
                operation.scrollable(state, id);
            }
            OperationWrapper::Id(operation) => {
                operation.scrollable(state, id);
            }
            OperationWrapper::Wrapper(operation) => {
                operation.scrollable(state, id);
            }
        }
    }

    fn text_input(&mut self, state: &mut dyn TextInput, id: Option<&Id>) {
        match self {
            OperationWrapper::Message(operation) => {
                operation.text_input(state, id);
            }
            OperationWrapper::Id(operation) => {
                operation.text_input(state, id);
            }
            OperationWrapper::Wrapper(operation) => {
                operation.text_input(state, id);
            }
        }
    }

    fn finish(&self) -> Outcome<OperationOutputWrapper<M>> {
        match self {
            OperationWrapper::Message(operation) => match operation.finish() {
                Outcome::None => Outcome::None,
                Outcome::Some(o) => {
                    Outcome::Some(OperationOutputWrapper::Message(o))
                }
                Outcome::Chain(c) => {
                    Outcome::Chain(Box::new(OperationWrapper::Message(c)))
                }
            },
            OperationWrapper::Id(operation) => match operation.finish() {
                Outcome::None => Outcome::None,
                Outcome::Some(id) => {
                    Outcome::Some(OperationOutputWrapper::Id(id))
                }
                Outcome::Chain(c) => {
                    Outcome::Chain(Box::new(OperationWrapper::Id(c)))
                }
            },
            OperationWrapper::Wrapper(c) => c.as_ref().finish(),
        }
    }
}
```
### addition of `a11y_nodes` method to `UserInterface`

each `UserInterface` needs to be able to produce an accessibility tree for its widget tree. It is simple to implement. Just call into `a11y_nodes` on the root widget.

### addition of `Event::A11y` variant

Widgets need some way of handling A11y events, so they are added as a variant in `Event`

```rust
#[cfg(feature = "a11y")]
/// An Accesskit event for a specific Accesskit Node in an accessible widget
A11y(
    iced_core::id::Id,
    iced_accessibility::accesskit::ActionRequest,
),
```

### Changes to iced_winit & iced_glutin

Changes to iced_winit are a superset of the changes to iced_glutin so I will only discuss the changes to iced_winit for now. They are mostly feature gated and include the following:

- Wrapper type around the winit event loop UserEvent
```rust
#[derive(Debug)]
/// Wrapper aroun application Messages to allow for more UserEvent variants
pub enum UserEventWrapper<Message> {
    /// Application Message
    Message(Message),
    #[cfg(feature = "a11y")]
    /// A11y Action Request
    A11y(iced_accessibility::accesskit_winit::ActionRequestEvent),
    /// A11y was enabled
    A11yEnabled,
}
```
- Adding an instance of the winit platform adapter
```rust
#[cfg(feature = "a11y")]
let (window_a11y_id, adapter) = {
    let node_id = iced_native::widget::id::window_node_id();

    use iced_accessibility::accesskit::{
        Node, NodeId, Role, Tree, TreeUpdate,
    };
    use iced_accessibility::accesskit_winit::Adapter;
    let title = state.title().to_string();
    let proxy_clone = proxy.clone();
    (
        node_id,
        Adapter::new(
            &window,
            move || {
                let _ =
                    proxy_clone.send_event(UserEventWrapper::A11yEnabled);
                TreeUpdate {
                    nodes: vec![(
                        NodeId(node_id),
                        std::sync::Arc::new(Node {
                            role: Role::Window,
                            name: Some(title.into_boxed_str()),
                            ..Default::default()
                        }),
                    )],
                    tree: Some(Tree::new(NodeId(node_id))),
                    focus: None,
                }
            },
            proxy.clone(),
        ),
    )
};
```
- adding a boolean that tracks whether accessibility has been enabled
- sending accessibility tree updates to accesskit if accessibility has been enabled
```rust
#[cfg(feature = "a11y")]
if a11y_enabled {
    use iced_accessibility::{
        accesskit::{Node, NodeId, Role, Tree, TreeUpdate},
        A11yId, A11yNode, A11yTree,
    };
    // TODO send a11y tree
    let child_tree = user_interface.a11y_nodes();
    let root = Node {
        role: Role::Window,
        name: Some(state.title().to_string().into_boxed_str()),
        ..Default::default()
    };
    let window_tree = A11yTree::node_with_child_tree(
        A11yNode::new(root, window_a11y_id),
        child_tree,
    );
    let tree = Tree::new(NodeId(window_a11y_id));
    let mut current_operation =
        Some(Box::new(OperationWrapper::Id(Box::new(
            operation::focusable::find_focused(),
        ))));
    let mut focus = None;
    while let Some(mut operation) = current_operation.take() {
        user_interface.operate(&renderer, operation.as_mut());

        match operation.finish() {
            operation::Outcome::None => {}
            operation::Outcome::Some(message) => match message {
                operation::OperationOutputWrapper::Message(
                    _,
                ) => {
                    unimplemented!();
                }
                operation::OperationOutputWrapper::Id(id) => {
                    focus = Some(A11yId::from(id));
                }
            },
            operation::Outcome::Chain(next) => {
                current_operation = Some(Box::new(
                    OperationWrapper::Wrapper(next),
                ));
            }
        }
    }

    adapter.update(TreeUpdate {
        nodes: window_tree.into(),
        tree: Some(tree),
        focus: focus.map(|id| id.into()),
    });
}
```
- reset widget Id counter before building the UserInterface

See https://github.com/wash2/iced for more implementation details

## Drawbacks

Drawbacks of this proposal are that the accessibility tree must be rebuilt every redraw. Accesskit also is still missing some widget implementations, though I'm not sure what they are. Lastly, the way focus is recalculated every redraw after accessibility is enabled may cause a performance hit depending on the application. There may also not be a 1:1 mapping from iced widget `Id` to accesskit `NodeId`, unless this kind of design is enforced in Iced. If a widget exists which maintains more than one accessibility node which may have application focus, the current proposal may not support it correctly. Ideally Though, each widget would contribute 1 Node to the accessibility tree. Maybe an additional variant of the iced Id type could include the Union of a set of Iced `Id`s and would match any of the `Id`s that it includes.


## Rationale and alternatives

- Why is this design the best in the space of possible designs?

This design is currently the only design, but it aims to neatly fit into the existing architecture and support incremental implementation.

- What other designs have been considered and what is the rationale for not choosing them?

I believe other designs might use a persistent widget tree and a diffing algorithm, which don't currently exist in Iced. Another option might 

- What is the impact of not doing this?

Iced would have to implement Accessibility some other way. If Iced does not support Accessibility, developers may choose another toolkit.

## Prior art

Egui supports accessibility via Accesskit. The Egui widget implementations could probably be used as a reference when implementing this proposal.


## Unresolved questions

- Maybe there should be a new Id variant?
- Maybe there should be more helper methods for the different widget types?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

Actual Widget implementations are somewhat out of scope of this RFC, as long as all existing widgets *can* be implemented using the skeleton proposed here.



## Future possibilities

- Persistent widget tree with diffing.
- Better focus handling
- automated testing of accessibility implementations
- multi-window (some changes to the winit implementation would need to be made)
