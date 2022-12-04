# Focus & Attributes
>>> ## Status: Discovery

## Summary

This proposal is for a shared attributes API. It is intended to be used by the focus system, but is not limited to that use case. It is also intended to be used by any system that needs to store or retrieve attributes on an element.

## Motivation

Why are we doing this? 
To support keyboard navigation and focus management in the runtime.

What use cases does it support? 
- Accessibility and keyboard navigation.
- Gamepad navigation.
- Element Attributes

What is the expected outcome?
As a developer I should have a similar experience to the browser runtime when it comes to focus management.

## Guide-level explanation

The goal is to design an API will be familiar to the developer and an a default experience that is an analogous the browser runtime.

By doing so we are more likely to meet the expectations of the developer. The browser runtime is the most common runtime for web developers and is has very robust and researched implementation.

> Explaining the feature largely in terms of examples.

### Example

The `ElementAttributes` holds a prescribed set of properties that can be used to determine the focus order of the element. The `ElementAttributes` also holds a `focusable` property that can be used to determine if the element is focusable or not.

We should make the implementation of the `ElementAttributes` as simple as possible. 
Could this be done automatically? Can we get a handle to the attributes from the widget if we did so?

```rs
    pub fn new(content: impl Into<Element<'a, Message, Renderer>>) -> Self {        
        Button {
            attributes: ElementAttributes::new(),
            content: content.into(),
            on_press: None,
            width: Length::Shrink,
            height: Length::Shrink,
            padding: Padding::new(5),
            style: <Renderer::Theme as StyleSheet>::Style::default(),
        }
    }
```
### Styling 

A default focus style would be applied to all focusable widgets. It can be overridden by the user if they want to by defining a custom style for the focused state.

To override the default styling we can use the `ElementAttributes` to access the attributes.

```rs
pub fn draw(
    &self,
    renderer: &mut Renderer,
    bounds: Rectangle,
    cursor_position: Point,
    viewport: &Rectangle,
) -> Renderer::Output {
    let is_mouse_over = bounds.contains(cursor_position);
    let is_focused = self.attributes.is_focused();

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

The basic idea is to place shared attributes in some sort of Mutex, or RwLock. This will allow us to share the attributes between the runtime and the widget. The runtime will be able to update the attributes and the widget will be able to read the attributes. Widgets or the Application should be able to query or update the attributes.

A cache should be implemented to to reduce locking and improve performance. The cache should be invalidated when the attributes is updated.


<!-- When the runtime is ready to render the widget it will be able to read the attributes to determine if the widget is focusable or not. If the widget is focusable the runtime will be able to determine the focus order of the widget. We should also compose a default focus style for the widget if one is not provided by the user. -->


The `ElementAttributes` could look something like this.

```rs
#[derive(Debug, Clone)]
pub struct ElementAttributes {
    id: Id,
    focusable: bool,
    focus_order: i32
}
```


The `ElementAttributesState` is stored in a `RwLock`.  The struct will look something like this. This attributes could expand in the future to support more accessibility features like ARIA. Or replace existing features like `hovered`.

```rs
pub struct ElementAttributesState {
    focused_id: Option<Id>,
    attributes: HashMap<Id, ElementAttributes>,
}

lazy_static!(
    pub static ref ELEMENT_ATTRIBUTES_STATE: ElementAttributesState = {
        ElementAttributesState {
            focused_id: None,
            attributes: HashMap::new(),
        }
    };
);
```


> Application Scope

- The application should be responsible for managing the focus. 
- The application should be responsible for setting the focus on the first element. 
- The application will also be responsible for setting the focus on the next element when the `Tab` key is pressed. 
- The application will also be responsible for setting the focus on the previous element when the `Shift + Tab` key is pressed.

> Query

The `ElementAttributes` will provide a other global methods that will allow us to close the gap between the browser runtime and iced. We should be able to query the attributes in the application or the widget. 

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

>Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:
> Does this feature exist in other GUI toolkits and what experience have their community had?

This feature exists in the browser runtime. This is the most familiar experience for developers.

> Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

Browser focus management is a complex topic. I have found the following resources to be helpful.

- [W3C Focus Order:](https://www.w3.org/TR/UNDERSTANDING-WCAG20/navigation-mechanisms-focus-order.html)
- [ARIA Developing a Keyboard Interface](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface)


## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

- Is the the direction we should be headed in? Is there a better solution? Is there a better way to implement this?

- How do we rerender the widget when the attributes changes? 

> What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

- Default styling for focus. We will need to set the default styling for focus if a focus style is not provided. This will allow us to provide a good experience out of the box.

- We may need to optimize the what the exact data structure will be for the shared state. And what strategies we will have the best characteristics for locking and caching the shared state.

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

- We should consider how to handle parent child relationships. This may be a future feature. We may need to always create a attributes handle for all widgets in the future.

- Hover state is also something that could be added to the attributes.

## [Optional] Future possibilities

Expanding on this concept to support more accesibility features like ARIA. 
Other widget attributes that could be handled by the runtime.

- Z-Index: The renderer could use this to determine the order of the widgets. This could be used to implement a stacking context.

- Visibility: This would also allow us to skip rendering and state updates if the widget is not visible. This would be a great performance improvement. Especially for large applications or infinite scroll applications. 

- Cached Bounds: This would allow us to cache the bounds of the widget. This would allow us to skip the layout pass if the widget is not visible. This would be a great performance improvement. Especially for large applications or infinite scroll applications.