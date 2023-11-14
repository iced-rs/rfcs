# Feature Name

## Summary

Implement focus for essential components to enable keyboard / gamepad / accessibility input navigation and interactions.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?
- Why?
    - To support nagivgating between focusable widgets.
- What use cases does it support?
    - Keyboard navigation and interactions
- What is the expected outcome?
    - As a user I should be able to navigate the widget tree with different stratagies 
        - Neighbor Strategy (Current Implementation)

## Guide-level explanation

Explain the proposal as if it was already included in the library and you were teaching it to another iced programmer. That generally means:

- Introducing new named concepts.

## The following widgets support focus based navigation and interactions
- text_input : When this widget has focus it will allow you to input text.
- button:
    - With focus it will 'click' with space, enter or num enter
    - Esc will clear focus
- pick_list:
    - With focus it will 'open' with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - With focus and is 'open' you may navigate the options with KeyCode::Up & KeyCode::Down
    - With focus and is 'open' you may select an option with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - Esc will close and clear focus
- toggler:
    - With focus it will 'toggle' with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - Esc will exit focus
- slider:
    - With focus it will toggle 'grab' with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - With focus and is 'grabbed' you may step the values the options with KeyCode::Left & KeyCode::Right (Or other directions if verticality is a implemented.)
    - Esc will ungrab and clear focus
- radio:
    - With focus it will 'select' current option with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - Esc will clear focus
- checkbox
    - With focus it will 'check' with KeyCode::Space, KeyCode::Enter, or KeyCode::NumpadEnter
    - Esc will clear focus
- scrollable
    - With focus it will 'scroll' with KeyCode::Space, KeyCode::PageDown, KeyCode::PageUp, KeyCode::Home ect.. 
    - (Questionable) If enabled scrollable will scroll to a child if the child gains focus and it is out of view. 



## Implementation strategy

Only one widget should have focus at a given time. At this time widgets are storing their own focus state.
Focus navigation is implemented based on this RFC and current implementation in text_input.

https://github.com/iced-rs/rfcs/blob/master/text/0007-widget-operations.md

https://github.com/iced-rs/iced/blob/master/native/src/widget/text_input.rs


## Drawbacks

The current implementation is using the path created by this RFC.
https://github.com/iced-rs/rfcs/blob/master/text/0007-widget-operations.md

The drawbacks to staying is that we can create unstable UI states by allowing multiple widgets to own their own focus states.
The drawbacks to moving to a global focus is that we will need to do more discovery and prolong the benefits of reaping the pattern we already have in place.

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
    - We should 
- What other designs have been considered and what is the rationale for not choosing them?
    - A global state design as discussed in this RFC would be ideal.
    - I am not convienced that we cannot maintain the current API for the most part and strangle each widgets internal focus state away to the global atomic. We would implement focus() functions that would interact with the global atomic focus_id.
- What is the impact of not doing this?
    - Staying as is means we can move forward with adding focus to the essential widgets.
    - Keeping multiple sources of truth will probably lead to buggy UX as more components are introduced into the system.


## [Optional] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other GUI toolkits and what experience have their community had?
    - Yes, Its a very important feature for critical adoption.

This section is intended to encourage you as an author to think about the lessons from other toolkits, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that iced sometimes intentionally diverges from common toolkit features.


## Unresolved questions

- Can we wait on global state in favor of enabling a critical feature for existing essential components?
- Ideally we could update the current implementation in a way that allows us to strangle the underlying details of global focus_id in a way with minimal or zero changes to the current exposed intergrations API.


## [Optional] Future possibilities

How would we implement a global state?

A global atomic id is used to track the current focused widget id?
All focusable widgets are given an atomic id.
```rs
static FOCUSED_ID: AtomicUsize = AtomicUsize::new(0);
```

Custom ids are not longer provided but retrieved. The application should keep its own map of atomics.
Custom string identifiers will return when `Once cell` is stable and `unsafe` is no longer required. 
```rs
iced::widget::id();
```

## global functions are available to change current focus
```rs
iced::widget::count() // a structure that has the current focus_id, and number of focusables 
iced::widget::focus(usize) // focus on a widget
iced::widget::unfocus() // release global focus as usize 0
```

Tab Style Navigation (Neighbor Strategy)
```rs
iced::widget::focus_next() // next sibling
iced::widget::focus_previous() // prev sibling
```

## Gamepad / Accessibility Navigation (Cardinal)
This strategy is based on screen space position of focusable widgets. A cardinal graph is built only for focusable widgets for fast navigation and avoid unesessary sqrt.

Gamepad & Accessibility navigation and interactions
- Keyboard navigation provides the lowest order implementation
- Emulated events could be sent to the enable accessiblity & gamepad interactions


## New global functions would be available to change current focus

```rs
iced::widget::focus_up() // up from current focus
iced::widget::focus_down() // down from current focus
iced::widget::focus_right() // right of current focus
iced::widget::focus_left() // left of current focus
```



## To support non-native devices like a gamepad or accessibility device these events can be emitted artifically.
- Keyboard(keyboard::Event)
- Mouse(mouse::Event)
- Touch(touch::Event)

```
No idea what this API needs to look like at this time.
```
