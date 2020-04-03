- Feature Name: 'Layers'
- Start Date: 2020-02-24

# Summary
[summary]: #summary

Implementing different layers of widgets so we can have overlays like drop down menus or dialogs.

# Motivation
[motivation]: #motivation

Right now widgets can only be laid out next to each other. There is no possibility to show a widget on top of another.
This proposal would allow widgets to declare their `Layout::Node` an overlay, so they wouldn't need to adhere to that.

# explanation
[explanation]: #explanation

First we add an `overlay` member to the `Layout::Node` struct, which is false by default. We also add a function to `Layout::Node`, that lets us determine the number of overlays in the node's tree:

```
pub fn overlay_order(&self) -> i32 {
    let order = if self.overlay { 1 } else { 0 };

    self.children
        .iter()
        .fold(order, |order, child| order + child.overlay_order())
}
```

This function is important to avoid overlay clashes by not putting two overlays into the same layer, but creating a new layer for every overlay.

That allows us to add a `layer` member to the `Layout` struct, which is 1 by default. Additionally we implement a new iterator that gets emitted by the `Layout::children()` function and looks like this:

```
struct LayoutIterator<Iter> {
    iter: Iter,
    parent_layer: i32,
    highest_layer: i32,
    offset: Vector,
}

impl<'a, Iter> LayoutIterator<Iter>
where
    Iter: Iterator<Item = &'a Node>,
{
    fn new(iter: Iter, parent_layer: i32, offset: Vector) -> Self {
        Self {
            iter,
            parent_layer,
            highest_layer: parent_layer,
            offset,
        }
    }
}

impl<'a, Iter> Iterator for LayoutIterator<Iter>
where
    Iter: Iterator<Item = &'a Node>,
{
    type Item = Layout<'a>;

    fn next(&mut self) -> Option<Self::Item> {
        self.iter.next().map(|node| {
            let mut layout = Layout::with_offset(self.offset, node);

            if node.overlay {
                layout.layer = self.highest_layer + 1;

                self.highest_layer += node.overlay_order();
            } else {
                layout.layer = self.parent_layer;
            };

            layout
        })
    }
}
```

This iterator takes care of setting the right `layer` of child layouts.

The `Widget` trait is then extended by a function that lets us determine the highest layer of the widget's `Layout` at a given position. The standard implementation of that function would look like this:

```
fn highest_layer_at(&self, position: Point, layout: Layout<'_>) -> i32 {
    if layout.bounds().contains(position) {
        layout.layer()
    } else {
        0
    }
}
```

Only widgets that contain other widgets like `Column`, `Row` or `Container` would need custom implementations.

This allows `UserInterface` to determine the highest layer under the mouse cursor by calling its root widget's `highest_layer_at` function with the mouse position as argument. It then passes that information along with the mouse position when calling `on_event` of its root widget.
All widgets need to be adapted to only react to mouse clicks and wheel events, if the highest layer under the cursor is equal to their own layer.
In any case the event should be propagated to any possible children, since there could be overlays inside a widgets subtree.

We also need to add a `Primitive::Overlay` variant to iced_wgpu. It just tells the renderer to draw everything inside its content member on top.
If the renderer encounters multiple overlay primitives, it draws them in the order they are added to the primitive tree.

# Drawbacks
[drawbacks]: #drawbacks

 - One more function widgets should implement.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we implement a new type for passing the highest layer along with the mouse position in `on_event` or pass the highest layer as an additional argument?
- Should every widget implement the event propagation logic itself or can we implement some of that in the `Widget` trait itself?
- Should the renderer clip overlays that are part of the content of a `Primitive::Clip`?

# Future possibilities
[future-possibilities]: #future-possibilities

- ComboBox widget
- Dialogs
- ContextMenu widget
- Tooltip widget

