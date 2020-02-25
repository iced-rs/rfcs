- Feature Name: 'Layers'
- Start Date: 2020-02-24

# Summary
[summary]: #summary

Giving widgets the ability to lie on top of each other and "capturing" events, preventing sibling and ancestor widgets to react to them.

# Motivation
[motivation]: #motivation

Right now widgets can only be layouted next to each other. There is no possibility to show a widget on top of another.
This proposal would allow us to assign a 'z' coordinate to widgets which determines the order in which they are drawn and events get propagated.

# Guide-level explanation
[explanation]: #explanation

The `Widget` trait is extended by a `z(&self) -> i16` function. The output of that function determines the "layer" a widget is drawn into compared to its siblings.
By default this function should return `1`.
Child widgets are still drawn on top of their parents and siblings with the same z-coordinate are drawn in the order they are pushed to their parent.
Event propagation to child widgets is prioritized by the children's z-coordinate. Children on top should receive events first.

# Drawbacks
[drawbacks]: #drawbacks

One more function widgets should implement.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should every widget implement the drawing and event propagation logic itself or can we implement some of that in the `Widget` trait itself?

# Future possibilities
[future-possibilities]: #future-possibilities

As a next step this functionality would allow us to create modal widgets.
These widgets would need to be children of the root widget with a higher z-coordinate than the rest of the root widget's children to be drawn on top of everything else and receive events first.
