- Feature Name: 'Layers'
- Start Date: 2020-02-24

# Summary
[summary]: #summary

Giving widgets the ability to lie on top of each other and "capturing" events, preventing sibling and ancestor widgets to react to them.

# Motivation
[motivation]: #motivation

Right now widgets can only be layouted next to each other. There is no possibility to show a widget on top of another.
This proposal would allow us to assign a 'z' coordinate to widgets which determines the order in which they are drawn and events get propagated.

# explanation
[explanation]: #explanation

The `Widget` trait is extended by a `z(&self) -> i16` function. The output of that function determines the "layer" a widget is drawn into compared to its siblings.
By default this function should return `1`.
Child widgets are still drawn on top of their parents and siblings with the same z-coordinate are drawn in the order they are pushed to their parent.
Since direct event propagation up the tree is impossible due to children not having a reference to their parent, we need to propagate them down the tree but each widget needs to propagate incoming events to its children in reverse order of their z-coordinate and after calling all children's `on_event' handler should the parent handle the event itself.
This way children drawn on top can "capture" events first and we simulate a propagate up effect.

As a next step this functionality would allow us to create modal widgets.
These widgets would need to be children of the root widget with a higher z-coordinate than the rest of the root widget's children to be drawn on top of everything else and receive events first.

# Drawbacks
[drawbacks]: #drawbacks

 - Two more functions widgets should implement.
 - Overlays need to be stateless.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should the root widget ask its descendants for overlays? A kind of child iterator would come in handy here.
- Should every widget implement the drawing and event propagation logic itself or can we implement some of that in the `Widget` trait itself?

# Future possibilities
[future-possibilities]: #future-possibilities


