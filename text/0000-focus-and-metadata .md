# First Class Focus

## Summary

Focus management is a complex topic and there are many different ways to handle it. This proposal is a possible solution that has closer parity with the browser runtime.

## Motivation

Why are we doing this? 
To support keyboard navigation and focus management in the runtime.

What use cases does it support? 
- Accessibility and keyboard navigation.
- Gamepad navigation.
- Element Metadata

What is the expected outcome?
As a developer I should have a simular experience to the browser runtime when it comes to focus management.

## Guide-level explanation

Explain the proposal as if it was already included in the library and you were teaching it to another iced programmer. That generally means:

> Introducing new named concepts.

The goal here is to remove the need for the `focus` method on the `Element` trait. This method is used to set the focus on a specific element. This is a very low level API and it is not very ergonomic. The new API will be more similar to the browser runtime.

By doing so we are more likely to meet the expectations of the developer. The browser runtime is the most common runtime for web developers. The browser runtime has a very good focus management system and we should try to match it. This may help provided a familar path if trying to implement WCAG compliance for iced applications.

To support this we need to be able to register element metadata. This metadata will be used to determine the focus order of the elements. This metadata will also be used to determine if an element is focusable or not.

> Explaining the feature largely in terms of examples.

The new API will be more similar to the browser runtime. This will allow developers to use the same patterns they are used to.


### Example

```rs
pub fn new(content: impl Into<Element<'a, Message, Renderer>>) -> Self {

    // Create a new metadata object. This will be used to determine the focus order internally a `MetadataHandle` is returned that can be used to access the metadata later.

    let metadata = ElementMetadata::new()
        .set_focusable(true)
        .set_focus_order(0);

    Button {
        metadata,
        content: content.into(),
        on_press: None,
        width: Length::Shrink,
        height: Length::Shrink,
        padding: Padding::new(5),
        style: <Renderer::Theme as StyleSheet>::Style::default(),
    }
}
```

When its time to draw the button we can use the `MetadataHandle` to access the metadata. This will allow us to determine if the button is focusable and what the focus order is.

```rs
pub fn draw(
    &self,
    renderer: &mut Renderer,
    bounds: Rectangle,
    cursor_position: Point,
    viewport: &Rectangle,
) -> Renderer::Output {
    let is_mouse_over = bounds.contains(cursor_position);
    let is_focused = self.metadata.is_focused();

    let mut styling = if !is_enabled {
        style_sheet.disabled(style)
    } else  {
        match (is_focused, is_hovered) {
            (true, false) => style_sheet.focused(style),
            (false, true) => style_sheet.hovered(style),
            (false, false) => style_sheet.active(style),
        }
    };
}
```

> Explaining how iced programmers should *think* about the feature, and how it should impact the way they use iced. It should explain the impact as concretely as possible.

We don't want to think about focus management. We want to be able to use the same patterns we are used to. This will allow us to focus on the application logic and not the focus management. Focus would be a first class citizen in iced.


## Implementation strategy

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

> Widget Scope

Internally we will use a shared state to determine the the current focus,focus order, and what to focus on next. The element metadata is accessible by the `MetadataHandle`. This will allow us to access the metadata from any thread. The `MetadataHandle` will be created in the `Widget`. This will allow us to access the metadata for drawing and logic.

When a `MetadataHandle` is created it will be added to a list of metadata handles. 

```rs
    // When we create the metadata it is also added to the list of metadata handles internally.
    let metadata = ElementMetadata::new()
        .set_focusable(true)
        .set_focus_order(0);
```

The `ElementMetadata` list and current Id will be stored in a `RwLock`.  The struct will look something like this. This metadata could expand in the future to support more accessibility features like ARIA. Or replace existing features like `hovered`.

```rs
struct ElementMetadata {
    id: Id,
    focusable: bool,    
    focus_order: u32,
    focus_next: Option<MetadataHandle>,
    focus_previous: Option<MetadataHandle>,
    focus: bool,
}
```

The list can be sorted by the focus order to to determine the focus order. The first element in the list will be the first element to receive focus. The last element in the list will be the last element to receive focus.

When a `MetadataHandle` is dropped it will be removed from the list of metadata handles. This will allow us to remove elements from the focus order.

The `MetadataHandle` methods will return cached values or utilize memoization. This will allow us to avoid locking for every access.

> Application Scope

- The application should be responsible for managing the focus. 
- The application should be responsible for setting the focus on the first element. - The application will also be responsible for setting the focus on the next element when the `Tab` key is pressed. The application will also be responsible for setting the focus on the previous element when the `Shift + Tab` key is pressed.

> Query

The `MetadataHandle` will provide a other global methods that will allow us to close the gap between the browser runtime and iced. We should be able to query the metadata in the application or the widget. 

## Drawbacks

The drawback of this proposal is that it follows the browser runtime. This may not be the best solution for iced. The browser runtime is not the best runtime for all applications. This may not be the best solution for all applications.

However there is nothing to stop someone from not using this API and implementing their own focus management system.

We should not do this if it is not possible to provide a good experience for the following use cases.
- Accessibility and keyboard navigation.
- Gamepad navigation.
- Developer Experience
- User Experience
- Performance
- Maintainability

## Rationale and alternatives

>  Why is this design the best in the space of possible designs?

It is the best design because it is the most similar to the browser runtime. This will allow developers to use the same patterns they are used to. The developer and the user will not have a jagged learning curve. This will allow developers to focus on the application logic and not the focus management. For the users of Iced applications they will have a friendly and familiar experiences out of the box.

> What other designs have been considered and what is the rationale for not choosing them?

In my previous RFC I attempted to solve the need of a global state via a global state management solution. Specifically one focused on UI state. The downside to that solution is it adds an additional state management solution to the mix. This will add complexity to the application. I think state management should be isolated to another initiative.

I also researched flutters `Focus Widget` based solution. As discussed in my previous RFC it is complex in nature and requires a proxy widget to wrap your widgets in.

> What is the impact of not doing this?

The impact of not doing this is that we will not be able to provide a good experience for the following use cases.

- Accessibility and keyboard navigation.
- Gamepad navigation.
- Developer Experience
- User Experience
- Maintainability


## [Optional] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

> Does this feature exist in other GUI toolkits and what experience have their community had?

This feature exists in the browser runtime. This is the most familiar experience for developers.

- Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

Browser focus management is a complex topic. I have found the following resources to be helpful.

- [W3C Focus Order:](https://www.w3.org/TR/UNDERSTANDING-WCAG20/navigation-mechanisms-focus-order.html)
- [ARIA Developing a Keyboard Interface](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface)

This section is intended to encourage you as an author to think about the lessons from other toolkits, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that iced sometimes intentionally diverges from common toolkit features.


## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

The API itself. We may need to add additional methods to the `MetadataHandle` to support more use cases. 

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

We may need to optimize the what the exact data structure will be for the shared state. And what strategies we will have the best characteristics for locking and caching the shared state.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

We should consider how to handle parent child relationships. This may be a future feature. We may need to always create a metadata handle for all widgets in the future.

Hover state is also something that could be added to the metadata.

## [Optional] Future possibilities

Expanding on this concept to support more accesibility features like ARIA. 
Other widget attributes that could be handled by the runtime.